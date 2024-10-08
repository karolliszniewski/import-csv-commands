1. Create a new module directory structure:

mkdir -p app/code/Custom/RemoveInvalidAttributes/Console/Command
mkdir -p app/code/Custom/RemoveInvalidAttributes/etc

2. Create the module.xml file:
vim app/code/Custom/RemoveInvalidAttributes/etc/module.xml

<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Custom_RemoveInvalidAttributes" setup_version="1.0.0">
    </module>
</config>

3. Create the di.xml file:
vim app/code/Custom/RemoveInvalidAttributes/etc/di.xml

<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Framework\Console\CommandList">
        <arguments>
            <argument name="commands" xsi:type="array">
                <item name="remove_invalid_attributes" xsi:type="object">Custom\RemoveInvalidAttributes\Console\Command\RemoveInvalidAttributes</item>
            </argument>
        </arguments>
    </type>
</config>

4. Create the command file:
vim app/code/Custom/RemoveInvalidAttributes/Console/Command/RemoveInvalidAttributes.php

<?php
namespace Custom\RemoveInvalidAttributes\Console\Command;

use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
use Magento\Framework\App\State;
use Magento\Catalog\Model\ResourceModel\Product\CollectionFactory;
use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Eav\Api\AttributeRepositoryInterface;
use Magento\Eav\Model\Config as EavConfig;
use Magento\Framework\App\ResourceConnection;

class RemoveInvalidAttributes extends Command
{
    protected $state;
    protected $productCollectionFactory;
    protected $productRepository;
    protected $attributeRepository;
    protected $eavConfig;
    protected $resource;

    public function __construct(
        State $state,
        CollectionFactory $productCollectionFactory,
        ProductRepositoryInterface $productRepository,
        AttributeRepositoryInterface $attributeRepository,
        EavConfig $eavConfig,
        ResourceConnection $resource
    ) {
        $this->state = $state;
        $this->productCollectionFactory = $productCollectionFactory;
        $this->productRepository = $productRepository;
        $this->attributeRepository = $attributeRepository;
        $this->eavConfig = $eavConfig;
        $this->resource = $resource;
        parent::__construct();
    }

    protected function configure()
    {
        $this->setName('custom:removeinvalidattributes')
             ->setDescription('Remove invalid attributes from products');
    }

    protected function execute(InputInterface $input, OutputInterface $output)
    {
        try {
            $this->state->setAreaCode(\Magento\Framework\App\Area::AREA_ADMINHTML);
            $productCollection = $this->productCollectionFactory->create();
            $connection = $this->resource->getConnection();

            foreach ($productCollection as $product) {
                $productId = $product->getId();
                $product = $this->productRepository->getById($productId);
                $attributes = $product->getAttributes();

                foreach ($attributes as $attribute) {
                    $attributeCode = $attribute->getAttributeCode();
                    $value = $product->getData($attributeCode);

                    if ($value === null || $value === '' || !$attribute->isValid($value)) {
                        try {
                            $attributeId = $attribute->getAttributeId();
                            $backendType = $attribute->getBackendType();
                            $tableName = $this->resource->getTableName("catalog_product_entity_$backendType");

                            $connection->delete(
                                $tableName,
                                [
                                    'attribute_id = ?' => $attributeId,
                                    'entity_id = ?' => $productId
                                ]
                            );

                            $output->writeln("Unlinked attribute '{$attributeCode}' from product ID {$productId}");
                        } catch (\Exception $e) {
                            $output->writeln("Error unlinking attribute '{$attributeCode}' from product ID {$productId}: " . $e->getMessage());
                        }
                    }
                }

                try {
                    $product->unsetData('_cache_instance_attributes');
                    $this->productRepository->save($product);
                    $output->writeln("Saved product ID {$productId}");
                } catch (\Exception $e) {
                    $output->writeln("Error saving product ID {$productId}: " . $e->getMessage());
                }
            }

            $output->writeln("Finished processing products.");
            return \Magento\Framework\Console\Cli::RETURN_SUCCESS;
        } catch (\Exception $e) {
            $output->writeln("<error>{$e->getMessage()}</error>");
            return \Magento\Framework\Console\Cli::RETURN_FAILURE;
        }
    }
}

5. Create the registration.php file:
vim app/code/Custom/RemoveInvalidAttributes/registration.php

<?php
\Magento\Framework\Component\ComponentRegistrar::register(
    \Magento\Framework\Component\ComponentRegistrar::MODULE,
    'Custom_RemoveInvalidAttributes',
    __DIR__
);
