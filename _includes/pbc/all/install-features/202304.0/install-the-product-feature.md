
This document describes how to integrate the Product feature into a Spryker project.

{% info_block infoBox "Included features" %}

The following feature integration guide expects the basic feature to be in place.
The current feature integration guide only adds the following functionalities:
* Product Concrete Search
* Add to cart from the Catalog page
* Discontinued product attributes postfix 

{% endinfo_block %}

This document describes how to integrate the Warehouse picking feature into a Spryker project.

## Install feature core

Follow the steps below to install the Product feature core.

### Prerequisites

Please overview and install the necessary features before beginning the integration step.

| Name | Version |
|---|---|
| Spryker Core | {{page.version}} |
| Prices | {{page.version}} |

### 1) Install the required modules using Composer

Run the following command to install the required modules:

```bash
composer require spryker-feature/product:"{{page.version}}" --update-with-dependencies
```
{% info_block warningBox "Verification" %}

Make sure that the following modules were installed:

| MODULE                       | EXPECTED DIRECTORY                      |
|------------------------------|-----------------------------------------|
| Product                      | spryker/product                         |
| ProductAttribute             | spryker/product-attribute               |
| ProductAttributeGui          | spryker/product-attribute-gui           |
| ProductCategoryFilter        | spryker/product-category-filter         |
| ProductCategoryFilterGui     | spryker/product-category-filter-gui     |
| ProductCategoryFilterStorage | spryker/product-category-filter-storage |
| ProductDiscontinuedStorage   | spryker/product-discontinued-storage    |
| ProductImageStorage          | spryker/product-image-storage           |
| ProductManagement            | spryker/product-management              |
| ProductPageSearch            | spryker/product-page-search             |
| ProductQuantityStorage       | spryker/product-quantity-storage        |
| ProductSearch                | spryker/product-search                  |
| ProductStorage               | spryker/product-storage                 |
| ProductSearchConfigStorage   | spryker/product-search-config-storage   |

{% endinfo_block %}


### 2) Set up database schema and transfer objects

Run the following commands to apply database changes and generate entity and transfer changes:

```bash
console transfer:generate
console propel:install
console transfer:generate
```

{% info_block warningBox "Verification" %}

Make sure that the following changes have been applied in transfer objects:

| TRANSFER | TYPE | EVENT | PATH |
| --- | --- | --- | --- |
| ProductImageFilter | class | created | src/Generated/Shared/Transfer/ProductImageFilterTransfer |
| ProductConcretePageSearch.images | property | added | src/Generated/Shared/Transfer/ProductConcretePageSearchTransfer |

{% endinfo_block %}

### 3) Add Translations

Append glossary according to your configuration:

**src/data/import/glossary.csv**

```yaml
quick-order.input.placeholder,Search by SKU or Name,en_US
quick-order.input.placeholder,Suche per SKU oder Name,de_DE
product_quick_add_widget.form.quantity,"# Qty",en_US
product_quick_add_widget.form.quantity,"# Anzahl",de_DE
quick-order.search.no_results,Item cannot be found,en_US
quick-order.search.no_results,Das produkt konnte nicht gefunden werden.,de_DE
product_search_widget.search.no_results,Products were not found.,en_US
product_search_widget.search.no_results,Products were not found.,de_DE
```

Run the following console command to import data:

```bash
console data:import glossary
```

{% info_block warningBox "Verification" %}

Make sure that the configured data is added to the `spy_glossary` table in the database.

{% endinfo_block %}

### 4) Set up behaviour

1. Enable the following behaviors by registering the plugins:

| PLUGIN                                       | SPECIFICATION                                                                                  | PREREQUISITES | NAMESPACE                                                       |
|----------------------------------------------|------------------------------------------------------------------------------------------------|---------------|-----------------------------------------------------------------|
| ProductViewDiscontinuedOptionsExpanderPlugin | Expands the product view super attributes and attribute variant map with discontinued postfix. |               | Spryker/Client/ProductDiscontinuedStorage/Plugin/ProductStorage |

**\Pyz\Zed\Oms\OmsDependencyProvider.php**

```php
<?php

namespace Pyz\Client\ProductStorage;

use Spryker\Client\ProductDiscontinuedStorage\Plugin\ProductStorage\ProductViewDiscontinuedOptionsExpanderPlugin;
use Spryker\Client\ProductStorage\ProductStorageDependencyProvider as SprykerProductStorageDependencyProvider;

class ProductStorageDependencyProvider extends SprykerProductStorageDependencyProvider
{
    /**
     * @return array<\Spryker\Client\ProductStorage\Dependency\Plugin\ProductViewExpanderPluginInterface>
     */
    protected function getProductViewExpanderPlugins(): array
    {
        $plugins = [
            new ProductViewDiscontinuedOptionsExpanderPlugin(),
        ];

        return $plugins;
    }
}
```

### 5) Set up configuration

Configure export to Redis and Elasticsearch.

Add to cart from catalog page configuration:

**src/Pyz/Zed/ProductPageSearch/ProductPageSearchConfig.php**

```php
<?php

namespace Pyz\Zed\ProductPageSearch;

use Spryker\Zed\ProductPageSearch\ProductPageSearchConfig as SprykerProductPageSearchConfig;

class ProductPageSearchConfig extends SprykerProductPageSearchConfig
{
    /**
     * @return bool
     */
    public function isProductAbstractAddToCartEnabled(): bool
    {
        return true;
    }
}
```

{% info_block warningBox "Verification" %}

Make sure that abstract products eligible for adding to cart have additional `add_to_cart_sku` field at Elasticsearch document.

{% endinfo_block %}

Configure discontinued product attributes postfix to have discontinued postfix for supper attributes only when the whole discontinued variant is selected.

**src/Pyz/Client/ProductDiscontinuedStorage/ProductDiscontinuedStorageConfig.php**

```php
<?php

namespace Pyz\Client\ProductDiscontinuedStorage;

use Spryker\Client\ProductDiscontinuedStorage\ProductDiscontinuedStorageConfig as SprykerProductDiscontinuedStorageConfig;

class ProductDiscontinuedStorageConfig extends SprykerProductDiscontinuedStorageConfig
{
    /**
     * @return bool
     */
    public function isOnlyDiscontinuedVariantAttributesPostfixEnabled(): bool
    {
        return true;
    }
}
```

{% info_block warningBox "Verification" %}

Make sure that product view super attributes and selected attributes are expanded with discontinued postfix only when the whole discontinued variant is selected.

{% endinfo_block %}

#### Product concrete search configuration

| PLUGIN | SPECIFICATION | PREREQUISITES | NAMESPACE |
| --- | --- | --- | --- |
| ProductConcretePageSearchProductImageEventSubscriber | Registers listeners that are responsible for publishing product concrete image entity changes to search when a related entity change event occurs. | None | Spryker\Zed\ProductPageSearch\Communication\Plugin\Event\Subscriber |
| ProductImageProductConcretePageMapExpanderPlugin | Expands product concrete page map with images field. | None | Spryker\Zed\ProductPageSearch\Communication\Plugin\PageMapExpander |
| ProductImageProductConcretePageDataExpanderPlugin | Expands product concrete page data with images data. | None | Spryker\Zed\ProductPageSearch\Communication\Plugin\PageMapExpander |

**src/Pyz/Zed/Event/EventDependencyProvider.php**

```php
<?php

namespace Pyz\Zed\Event;

use Spryker\Zed\Event\EventDependencyProvider as SprykerEventDependencyProvider;
use Spryker\Zed\ProductPageSearch\Communication\Plugin\Event\Subscriber\ProductConcretePageSearchProductImageEventSubscriber;

class EventDependencyProvider extends SprykerEventDependencyProvider
{
    public function getEventSubscriberCollection()
    {
        $eventSubscriberCollection = parent::getEventSubscriberCollection();
        $eventSubscriberCollection->add(new ProductConcretePageSearchProductImageEventSubscriber());

        return $eventSubscriberCollection;
    }
}
```

**src/Pyz/Zed/ProductPageSearch/ProductPageSearchDependencyProvider.php**

```php
<?php

namespace Pyz\Zed\ProductPageSearch;

use Spryker\Zed\ProductPageSearch\Communication\Plugin\PageDataExpander\ProductImageProductConcretePageDataExpanderPlugin;
use Spryker\Zed\ProductPageSearch\Communication\Plugin\PageMapExpander\ProductImageProductConcretePageMapExpanderPlugin;
use Spryker\Zed\ProductPageSearch\ProductPageSearchDependencyProvider as SprykerProductPageSearchDependencyProvider;

class ProductPageSearchDependencyProvider extends SprykerProductPageSearchDependencyProvider
{
    /**
     * @return \Spryker\Zed\ProductPageSearchExtension\Dependency\Plugin\ProductConcretePageMapExpanderPluginInterface[]
     */
    protected function getConcreteProductPageMapExpanderPlugins(): array
    {
        return [
            new ProductImageProductConcretePageMapExpanderPlugin(),
        ];
    }

    /**
     * @return \Spryker\Zed\ProductPageSearchExtension\Dependency\Plugin\ProductConcretePageDataExpanderPluginInterface[]
     */
    protected function getProductConcretePageDataExpanderPlugins(): array
    {
        return [
            new ProductImageProductConcretePageDataExpanderPlugin(),
        ];
    }
}
```

{% info_block warningBox "Verification" %}

1. Make sure that after `console sync:data product_concrete` command execution product data including images is synced to Elasticsearch product concrete documents.
2. Also check that when product or its images updated via Zed UI, product data including images is synced at respective Elasticsearch product concrete documents.

| STORAGE TYPE | TARGET ENTITY | EXAMPLE EXPECTED DATA IDENTIFIER |
| --- | --- | --- |
| Elasticsearch | ProductConcrete | product_concrete:de:de_de:1 |

**Example expected data fragment**

```yaml
{
   "store":"DE",
   "locale":"de_DE",
   "type":"product_concrete",
   "is-active":true,
   "search-result-data":{
      "id_product":1,
      "fkProductAbstract":2,
      "abstractSku":"222",
      "sku":"222_111",
      "type":"product_concrete",
      "name":"HP 200 280 G1",
      "images":[
         {
            "idProductImage":1,
            "idProductImageSetToProductImage":1,
            "sortOrder":0,
            "externalUrlSmall":"//images.icecat.biz/img/gallery_mediums/img_29406823_medium_1480596185_822_26035.jpg",
            "externalUrlLarge":"//images.icecat.biz/img/gallery_raw/29406823_8847.png"
         }
      ]
   },
   "full-text-boosted":[
      "HP 200 280 G1",
      "222_111"
   ],
   "suggestion-terms":[
      "HP 200 280 G1",
      "222_111"
   ],
   "completion-terms":[
      "HP 200 280 G1",
      "222_111"
   ],
   "string-sort":{
      "name":"HP 200 280 G1"
   }
}
```
{% endinfo_block %}


## Install feature frontend

### Prerequisites

Overview and install the necessary features before beginning the integration step.

| Name | Version |
| --- | --- |
| Spryker Core | {{page.version}} |

### 1) Install the required modules using Composer

Run the following command to install the required modules:

```bash
composer require spryker-feature/product:"{{page.version}}" --update-with-dependencies
```

{% info_block warningBox "Verification" %}

Make sure that the following modules are installed:

| MODULE | EXPECTED DIRECTORY |
| --- | --- |
| ProductSearchWidget | spryker-shop/product-search-widget |

{% endinfo_block %}

### 2) Set up widgets

Register the following plugins to enable widgets:

| PLUGIN | SPECIFICATION | PREREQUISITES | NAMESPACE |
| --- | --- | --- | --- |
| ProductConcreteSearchWidget | Allows customers to search for concrete products on the Cart page. | None | SprykerShop\Yves\ProductSearchWidget\Widget |
| ProductConcreteSearchWidget | Incorporates `ProductConcreteSearchWidget` and allows customers to search for concrete products and quickly add them to the Cart with the desired quantity. | None | SprykerShop\Yves\ProductSearchWidget\Widget |
| ProductConcreteSearchGridWidget | Allows to output list of concrete products from search filtered by criteira. | None | SprykerShop\Yves\ProductSearchWidget\Widget |

**src/Pyz/Yves/ShopApplication/ShopApplicationDependencyProvider.php**

```php
<?php

namespace Pyz\Yves\ShopApplication;

use SprykerShop\Yves\ProductSearchWidget\Widget\ProductConcreteAddWidget;
use SprykerShop\Yves\ProductSearchWidget\Widget\ProductConcreteSearchWidget;
use SprykerShop\Yves\ProductSearchWidget\Widget\ProductConcreteSearchGridWidget;
use SprykerShop\Yves\ShopApplication\ShopApplicationDependencyProvider as SprykerShopApplicationDependencyProvider;

class ShopApplicationDependencyProvider extends SprykerShopApplicationDependencyProvider
{
    /**
     * @return string[]
     */
    protected function getGlobalWidgets(): array
    {
        return [
            ProductConcreteSearchWidget::class,
            ProductConcreteAddWidget::class,
            ProductConcreteSearchGridWidget::class,
        ];
    }
}
```

{% info_block warningBox "Verification" %}

Make sure that the following widgets are registered:

| MODULE | TEST |
| --- | --- |
| ProductConcreteSearchWidget | Go to the Cart page and make sure the "Quick add to Cart" section is present, so you can search for concrete products by typing their SKU. |
| ProductConcreteAddWidget | Go to the Cart page and make sure the "Quick add to Cart" section is present, so you can add the found products to the Cart with the desired Quantity. |
| ProductConcreteSearchGridWidget | Could be checked on slot edit page of Configurator. |

{% endinfo_block %}