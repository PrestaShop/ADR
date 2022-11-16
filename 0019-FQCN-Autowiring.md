# 19. FQCN & autowiring

Date: 2022-11-16

## Status

Approved

## Context

As PrestaShop 8.0 as been released, PrestaShop is still using snake case for services names. The proposition is to use service FQCN for services and enable use of autowiring. This is [the recommended way](https://symfony.com/doc/current/contributing/code/standards.html#service-naming-conventions) from Symfony.

That allows usage of this features 

- Simpler services configuration (improves DX as well).
- Controller service arguments can be used
- Autowiring
- Container optimization (as we can update and remove badly configured public services)

If there are some interfaces that are used many times, injection can still be done manually.


## Consequences

- This notation will become the default one for all service, unless for exceptional specific uses cases.
- We will need to rename all services to avoid confusion so they use FQCNs. That can be done gradually.
  - Create new private service definitions for these services
  - Update the old definition to set it as an alias of the new service
  - If the old service was public, set the alias as `public: true`
  - Deprecate the old service and it will be removed in next major.
  - We need to ensure the decoration still works for deprecated aliases for third party modules.

Example: 

```yaml
# from
prestashop.bundle.my_service_name:
  alias: PrestaShopBundle\ServiceName
  public : true
  deprecated: ~

# to
PrestaShopBundle\ServiceName:
  autowire: true
  public: false # implicit
```
