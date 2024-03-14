# Shopware API Client

A Django-ORM like, Python 3.12, async Shopware 6 admin and store-front API client.

## Installation

pip install shopware-api-client

## Usage

There are two kinds of clients provided by this library. The `client.AdminClient` for the Admin API and the
`client.StoreClient` for the Store API.

### client.AdminClient

To use the AdminClient you need to create a `config.AdminConfig`. The `AdminConfig` class supports two login methods
(grant-types):
- **client_credentials** (Default) Let's you log in with a `client_id` and `client_secret`
- **password** Let's you log in using a `username` and `password`

You also need to provide the Base-URL of your shop.

Example:

```python
from shopware_api_client.config import AdminConfig

CLIENT_ID = "MyClientID"
CLIENT_SECRET = "SuperSecretToken"
SHOP_URL = "https://pets24.shop"

config = AdminConfig(url=SHOP_URL, client_id=CLIENT_ID, client_secret=CLIENT_SECRET, grant_type="client_credentials")

# or for "password"

ADMIN_USER = "admin"
ADMIN_PASSWORD = "!MeowMoewMoew~"

config = AdminConfig(url=SHOP_URL, username=ADMIN_USER, password=ADMIN_PASSWORD, grant_type="password")
```

Now you can create the Client. There are two output formats for the client, that can be selected by the `raw` parameter:
- **raw=True** Outputs the result as a plain dict or list of dicts
- **raw=False** (Default) Outputs the result as Pydantic-Models

```python
from shopware_api_client.client import AdminClient

# Model-Mode
client = AdminClient(config=config)

# raw-Mode
client = AdminClient(config=config, raw=True)
```

Client-Connections should be closed after usage: `await client.close()`. The client can also be used in an `async with`
block to be closed automatically.

```python
from shopware_api_client.client import AdminClient

async with AdminClient(config=config) as client:
    # do important stuff
    pass
```

All registered Endpoints are directly available from the client instance. For example if you want to query the Customer
Endpoint:

```python
customer = await client.customer.first()
```

All available Endpoint functions can be found in the [EndpointBase](#list-of-available-functions) section.

There are two additional ways how the client can be utilized by using it with the Endpoint-Class directly or the
associated Pydantic Model:

```python
from shopware_api_client.endpoints.admin.core.customer import Customer, CustomerEndpoint

# Endpoint
customer_endpoint = CustomerEndpoint(client=client)
customer = await customer_endpoint.first()

# Pydantic Model
customer = await Customer.using(client=client).first()
```

#### Related Objects

If you use the Pydantic-Model approach (`raw=False`) you can also use the returned object to access its related objects:

```python
from shopware_api_client.endpoints.admin import Customer

customer = await Customer.using(client=client).first()
customer_group = await customer.group  # Returns a CustomerGroup object
all_the_customers = await customer_group.customers  # Returns a list of Customer objects
```

**!! Be careful to not close the client before doing related objects calls, since they use the same Client instance !!**
```python
from shopware_api_client.client import AdminClient
from shopware_api_client.endpoints.admin import Customer

async with AdminClient(config=config) as client:
    customer = await Customer.using(client=client).first()

customer_group = await customer.group  # This will fail, because the client connection is already closed!
```

### client.StoreClient

To use the StoreClient you need to create a `config.StoreConfig`. The `StoreConfig` needs a store api access key.
You also need to provide the Base-URL of your shop.

Some Endpoints (that are somehow related to a user) require a context-token. This parameter is optional.

Example:

```python
from shopware_api_client.config import StoreConfig

ACCESS_KEY = "SJMSAKSOMEKEY"
CONTEXT_TOKEN = "ASKSKJNNMMS"
SHOP_URL = "https://pets24.shop"

config = StoreConfig(url=SHOP_URL, access_key=STORE_API_ACCESS_KEY, context_token=CONTEXT_TOKEN)
```

This config can be used with the `StoreClient`, which works exactly like the `AdminClient`.

## EndpointBase
The `base.EndpointBase` class should be used for creating new Endpoints. It provides some usefull functions to call
the Shopware-API.

The base structure of an Endpoint is pretty simple:

```python
from shopware_api_client.base import ApiModelBase, EndpointBase
from shopware_api_client.client import registry

class CustomerGroup(ApiModelBase["CustomerGroupEndpoint"]):
    # Model definition
    pass

class CustomerGroupEndpoint(EndpointBase[CustomerGroup]):
    name = "customer_group"  # name of the Shopware-Endpoint (snaky)
    path = "/customer-group"  # path of the Shopware-Endpoint
    model_class = CustomerGroup  # Pydantic-Model of this Endpoint

registry.register_admin(CustomerGroupEndpoint)  # Register it so it will be added to the client
```

### List of available Functions

- `all()` return all objects (GET /customer-group or POST /search/customer-group if filter or sort is set)
- `get(pk: str = id)` returns the object with the passed id (GET /customer-group/id)
- `update(pk: str = id, obj: ModelClass | dict[str: Any]` updates an object (PATCH /customer-group/id)
- `create(obj: ModelClass | dict[str: Any]` creates a new object (POST /customer-group)
- `delete(pk: str = id)` deletes an object (DELETE /customer-group/id)
- `filter(name="Cats")` adds a filter to the query. Needs to be called with .all(), .iter() or .first())) More Info: [Filter](#filter)
- `limit(count: int | None)` sets the limit parameter, to limit the amount of results. Needs to be called with .all() or .first()
- `first()` sets the limit to 1 and returns the result (calling .all())
- `order_by(fields: str | tuple[str]` sets the sort parameter. Needs to be called with .all(), .iter() or .first(). Syntax: "name" for ASC, "-name" for DESC
- `iter(batch_size: int = 100)` sets the limit-parameter to batch_size and makes use of the pagination of the api. Should be used when requesting a big set of data (GET /customer-group or POST /search/customer-group if filter or sort is set)
- `bulk_upsert(objs: list[ModelClass] | list[dict[str, Any]` creates/updates multiple objects. Does always return dict of plain response. (POST /_action/sync)
- `bulk_delete(objs: list[ModelClass] | list[dict[str, Any]` deletes multiple objects. Does always return dict or plain response. (POST /_action/sync)

Not all functions are available for the StoreClient-Endpoints. But some of them have some additional functions.

### Filter

The `filter()` functions allows you to filter the result of an query. The parameters are basically the field names.
You can add an appendix to change the filter type. Without it looks for direct matches (equals). The following
appendices are available:

- `__in` expects a list of values, matches if the value is provided in this list (equalsAny)
- `__contains` matches values that contain this value (contains)
- `__gt` greater than (range)
- `__gte` greater than equal (range)
- `__lt` lower than (range)
- `__lte` lower than equal (range)
- `__range` expects a touple of two items, matches everything inbetween. inclusive. (range)
- `__startswith` matches if the value starts with this (prefix)
- `__endswith` matches if the value ends with this (suffix)

For some fields (that are returned as dict, like custom_fields) it's also possible to filter over the values of it's
keys. To do so you can append the key seperated by "__" For example if we have a custom field called "preferred_protein"
we can filter on it like this:
```python
customer = await Customer.using(client=client).filter(custom_field__preferred_protein="fish")

# or with filter-type-appendix
customer = await Customer.using(client=client).filter(custom_field__preferred_protein__in=["fish", "chicken"])
```

## ApiModelBase

The `base.ApiModelBase` class is basicly a `pydantic.BaseModel` which should be used to create Endpoint-Models.

The base structure of an Endpoint-Model looks like this:

```python
from pydantic import Field
from typing import Any
from shopware_api_client.base import ApiModelBase

class CustomerGroup(ApiModelBase["CustomerGroupEndpoint"]):
    _identifier = "customer_group"  # name of the Shopware-Endpoint (snaky)

    name: str  # Field with type
    display_gross: bool | None = Field(default=None, alias="displayGross")  # for fields that do not look snaky we need to provide an alias
    custom_fields: dict[str, Any] | None = Field(default=None, alias="customFields")
    # other fields...
```

The `id` attribute is provided in the ApiModelBase and must not be added.

### List of available Function

- `save()` executes `Endpoint.update()` if an id is set otherwise `Endpoint.create()`
- `delete()` executes `Endpoint.delete()`


### Relations

To make relations to other models work, we have to define them in the Model. There are two classes to make this work:
`endpoints.relations.ForeignRelation` and `endpoints.relations.ManyRelation`.

- `ForeignRelation(model: str, source: str)` is used when we get the id of the related object in the api response.
  - `model`: Name of the related Model as string
  - `source`: Name of the field that contains the id

- `ManyRelation(model: str, api_link: str)` is used for the reverse relation. We don't get ids in the api response, but it can be used through
relation links.
  - `model`: Name of the related Model as string
  - `api_link`: URL appendix of the Shopware API is using to get this relation. Provided as relations-link in the API.
  Example: https://pets24.shop/api/customer/someCustomerId/addresses lets you get all CustomerAddress Objects related
  to this Customer. `api_link` would be "addresses" here.

Example (Customer):
```python
from pydantic import constr, Field
from typing import Awaitable, ClassVar, TYPE_CHECKING

from shopware_api_client.base import ApiModelBase, EndpointClass
from shopware_api_client.endpoints.relations import ForeignRelation, ManyRelation

if TYPE_CHECKING:
    from shopware_api_client.endpoints.admin import CustomerAddress

# Base-Class for the normal model fields
class CustomerBase(ApiModelBase[EndpointClass]):
    # we have an id so we can create a ForeignRelation to it
    default_billing_address_id: constr(pattern=r'^[0-9a-f]{32}$') = Field(alias="defaultBillingAddressId")


# Relations-Class for the related fields
class CustomerRelations:
    # ClassVar: We need to use ClassVar to make Pydantic ignore this
    default_billing_address: ClassVar[ForeignRelation["CustomerAddress"]] = ForeignRelation(
      "CustomerAddress", "default_billing_address_id"
    )

    # We don't have a field for all addresses of a customer, but there is a relation for it!
    addresses: ClassVar[ManyRelation["CustomerAddress"]] = ManyRelation("CustomerAddress", "addresses")


# Final Class, that combines both of them
class Customer(CustomerBase["CustomerEndpoint"], CustomerRelations):
    pass
```

We have two classes `Base` and `Relations`. This way we can [reuse the Base-Model](#reusing-admin-models-for-store-endpoints).

## Development

### Model Creation

Shopware provides API-definitions for their whole API. You can download it from `<shopurl>/api/_info/openapi3.json`
Then you can use tools like `datamodel-code-generator`

```
datamodel-codegen --input openapi3.json --output model_openapi3.py --snake-case-field --use-double-quotes --output-model-type=pydantic_v2.BaseModel --use-standard-collections --use-union-operator
```

The file may look confusing at first, but you can search for Endpoint-Name + JsonApi (Example: class CustomerJsonApi)
to get all returned fields + relationships class as an overview over the available Relations. However, the Models will
need some Modifications. But it's a good start.

Not all fields returned by the API are writeable and the API will throw an error when you try to set it. So this fields
must have an `exclude=True` in their definition. To find out which fields need to be excluded check the Shopware
Endpoint documentation at https://shopware.stoplight.io/. Go to the Endpoint your Model belongs to and check the
available POST fields.

The newly created Model must than be imported to `admin/__init__.py` or `store/__init__.py` and added to `__all__`.
This import is necessary for the registry registration to be executed. The `__all__` statement is necessary so they
don't get cleaned away as unused imports by code-formaters/cleaners.

### Step by Step Example (Admin Endpoint Media Thumbnail)

1. Create the file for the endpoint. Since Media Thumbnail is an Admin > Core Endpoint we create a file called `media_thumbnail.py` in `endpoints/admin/core/`

2. You can copy & paste the following example as a base for the new Endpoint:

```python
from typing import TYPE_CHECKING, Any, ClassVar

from pydantic import AwareDatetime, Field

from ....base import ApiModelBase, EndpointBase, EndpointClass
from ....client import registry
from ...base_fields import IdField
from ...relations import ForeignRelation, ManyRelation

if TYPE_CHECKING:
    from ...admin import ForeignRelationModel, ManyRelationModel


class YourModelBase(ApiModelBase[EndpointClass]):
    _identifier = "your_model"

    foreign_id: IdField = Field(..., alias="foreignId")
    # more direct fields


class YourModelRelations:
    foreign: ClassVar[ForeignRelation["ForeignRelationModel"]] = ForeignRelation("ForeignRelationModel", "foreign_id")
    many: ClassVar[ManyRelation["ManyRelationModel"]] = ManyRelation("ManyRelationModel", "many-relation")


class YourModel(YourModelBase["YourModelEndpoint"], YourModelRelations):
    pass


class YourModelEndpoint(EndpointBase[YourModel]):
    name = "your_model"
    path = "/your-model"
    model_class = YourModel


registry.register_admin(YourModelEndpoint)

```

3. Update the example to your needs (Media Thumbnail Example):
    * Replace `YourModel` with `MediaThumbnail`
    * Replace `your_model` with `media_thumbnail`
    * Replace `your-model` with `media-thumbnail`

4. Assuming you used the datamodel-codegen command above to generate datamodels you can search the file for
`class MediaThumbnailJsonApi` and copy all fields except `id` (included in ApiModelBase) and `relationships`.
ID-Fields will use the type `constr(pattern=r"^[0-9a-f]{32}$")`. Replace it with `IdField` from `endpoints.base_fields`.

5. Now your `media_thumbnail.py` should look like this:

```python
from typing import TYPE_CHECKING, Any, ClassVar

from pydantic import AwareDatetime, Field

from ....base import ApiModelBase, EndpointBase, EndpointClass
from ....client import registry
from ...base_fields import IdField
from ...relations import ForeignRelation, ManyRelation

if TYPE_CHECKING:
    from ...admin import ForeignRelationModel, ManyRelationModel


class MediaThumbnailBase(ApiModelBase[EndpointClass]):
    _identifier = "media_thumbnail"

    media_id: IdField = Field(..., alias="mediaId")
    width: int
    height: int
    url: str | None = Field(
        None, description="Runtime field, cannot be used as part of the criteria."
    )
    path: str | None = None
    custom_fields: dict[str, Any] | None = Field(None, alias="customFields")
    created_at: AwareDatetime = Field(..., alias="createdAt")
    updated_at: AwareDatetime | None = Field(None, alias="updatedAt")


class MediaThumbnailRelations:
    foreign: ClassVar[ForeignRelation["ForeignRelationModel"]] = ForeignRelation("ForeignRelationModel", "foreign_id")
    many: ClassVar[ManyRelation["ManyRelationModel"]] = ManyRelation("ManyRelationModel", "manyRelation")


class MediaThumbnail(MediaThumbnailBase["MediaThumbnailEndpoint"], MediaThumbnailRelations):
    pass


class MediaThumbnailEndpoint(EndpointBase[MediaThumbnail]):
    name = "media_thumbnail"
    path = "/media-thumbnail"
    model_class = MediaThumbnail


registry.register_admin(MediaThumbnailEndpoint)
```

6. Next up: Relations. For this check the type of the `relationships`. `MediaThumbnail` has only one relation to `media`.
If you follow the types of `media > data` you can see the actual model type in the `type` field as examples attribute: media.
So it relates to the Media Endpoint. It's a ForeignRelation the object related in our `media_id` field.
For this relation our related Field looks like this:
`media: ClassVar[ForeignRelation["Media"]] = ForeignRelation("Media", "media_id")`
We add it to the Relations class.

7. Updated `media_thumbnails.py`:
```python
from typing import TYPE_CHECKING, Any, ClassVar

from pydantic import AwareDatetime, Field

from ....base import ApiModelBase, EndpointBase, EndpointClass
from ....client import registry
from ...base_fields import IdField
from ...relations import ForeignRelation

if TYPE_CHECKING:
    from ...admin import Media


class MediaThumbnailBase(ApiModelBase[EndpointClass]):
    _identifier = "media_thumbnail"

    media_id: IdField = Field(..., alias="mediaId")
    width: int
    height: int
    url: str | None = Field(
        None, description="Runtime field, cannot be used as part of the criteria."
    )
    path: str | None = None
    custom_fields: dict[str, Any] | None = Field(None, alias="customFields")
    created_at: AwareDatetime = Field(..., alias="createdAt")
    updated_at: AwareDatetime | None = Field(None, alias="updatedAt")


class MediaThumbnailRelations:
    media: ClassVar[ForeignRelation["Media"]] = ForeignRelation("Media", "media_id")


class MediaThumbnail(MediaThumbnailBase["MediaThumbnailEndpoint"], MediaThumbnailRelations):
    pass


class MediaThumbnailEndpoint(EndpointBase[MediaThumbnail]):
    name = "media_thumbnail"
    path = "/media-thumbnail"
    model_class = MediaThumbnail


registry.register_admin(MediaThumbnailEndpoint)
```

8. Now we have to check, which fields are read-only fields. The easiest way for this is to head to the POST section of the documentation of this Endpoint: https://shopware.stoplight.io/docs/admin-api/9724c473cce7d-create-a-new-media-thumbnail-resources
All fields that aren't listed here are read-only fields. So for our example this are: width, height, path, created_at and updated_at. We need to add an `exclude=True` to this fields, to make Pydantic ignore this fields when we send them back to the
API for saving or creating entries.

9. After adding `exclude=True` our final file should look like this:
```python
from typing import TYPE_CHECKING, Any, ClassVar

from pydantic import AwareDatetime, Field

from ....base import ApiModelBase, EndpointBase, EndpointClass
from ....client import registry
from ...base_fields import IdField
from ...relations import ForeignRelation

if TYPE_CHECKING:
    from ...admin import Media


class MediaThumbnailBase(ApiModelBase[EndpointClass]):
    _identifier = "media_thumbnail"

    media_id: IdField = Field(..., alias="mediaId")
    width: int = Field(..., exclude=True)
    height: int = Field(..., exclude=True)
    url: str | None = Field(
        None, description="Runtime field, cannot be used as part of the criteria."
    )
    path: str | None = Field(None, exclude=True)
    custom_fields: dict[str, Any] | None = Field(None, alias="customFields")
    created_at: AwareDatetime = Field(..., alias="createdAt", exclude=True)
    updated_at: AwareDatetime | None = Field(None, alias="updatedAt", exclude=True)


class MediaThumbnailRelations:
    media: ClassVar[ForeignRelation["Media"]] = ForeignRelation("Media", "media_id")


class MediaThumbnail(MediaThumbnailBase["MediaThumbnailEndpoint"], MediaThumbnailRelations):
    pass


class MediaThumbnailEndpoint(EndpointBase[MediaThumbnail]):
    name = "media_thumbnail"
    path = "/media-thumbnail"
    model_class = MediaThumbnail


registry.register_admin(MediaThumbnailEndpoint)
```
10. As final step we need to add an import for the Model to `endpoints/admin/__init__.py` and add the Model to `__all__`
```python
# other imports
from .core.admin.media_thumbnail import MediaThumbnail
# more imports

__all__ = [
  # other models
  "MediaThumbnail",
  # more models
]
```

We are done and you are now ready to use your new endpoint! 🎉

### Reusing Admin-Models for Store-Endpoints

The Store-Endpoints use the same Model structure as the Admin-Endpoints, but have no relations. Some of the related
objects are added to the response directly. We can use the Base-Models from our Admin-Endpoints for this purpose:

```python
from ...admin.core.country import CountryBase
from ...admin.core.country_state import CountryStateBase
from ...admin.core.customer_address import CustomerAddressBase
from ...admin.core.salutation import SalutationBase


class Address(CustomerAddressBase["AddressEndpoint"]):
    _identifier = "address"

    country: CountryBase | None = None
    customer_state: CountryStateBase | None = Field(None, alias="countryState")
    salutation: SalutationBase | None = None
```

Since country, customerState and salutation are returned in the response, we can use their Base-Models to define
their types.

This "related" fields should always be optional, because their value is not always returned (object-creation).

### Structure

```
> endpoints  -- All endpoints live here
  > admin  -- AdminAPI endpoints
    > core  -- AdminAPI > Core
      customer_address.py  -- Every Endpoint has its own file. Model and Endpoint are defined here
    > commercial  -- AdminAPI > Commercial
    > digital_sales_rooms  -- AdminAPI > Digital Sales Rooms
  > store  -- StoreAPI
    > core  -- StoreAPI > Core
    > commercial  -- StoreAPI > Commercial
    > digital_sales_rooms  -- StoreAPI > Digital Sales Rooms
base.py  -- All the Base Classes
client.py  -- Clients & Registry
config.py  -- Configs
exceptions.py  -- Exceptions
logging.py  -- Logging
tests.py  -- tests
```
