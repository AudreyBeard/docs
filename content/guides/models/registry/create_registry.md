---
menu:
  default:
    identifier: create_registry
    parent: registry
title: Create a custom registry
weight: 2
---

Create a [custom registry]({{< relref "./registry_types.md#custom-registry" >}}) for each step of your ML workflow.

Custom registries are particularly useful for organizing project-specific requirements that differ from the default, [core registry]({{< relref "./registry_types.md#core-registry" >}}).

The following procedure describes how to interactively create a registry:
1. Navigate to the **Registry** App in the W&B App UI.
2. Within **Custom registry**, click on the **Create registry** button.
3. Provide a name for your registry in the **Name** field.
4. Optionally provide a description about the registry.
5. Select who can view the registry from the **Registry visibility** dropdown. See [Registry visibility types]({{< relref "./configure_registry.md#registry-visibility-types" >}}) for more information on registry visibility options.
6. Select either **All types** or **Specify types** from the **Accepted artifacts type** dropdown.
7. (If you select **Specify types**) Add one or more artifact types that your registry accepts.
{{% alert %}}
An artifact type can not be removed from a registry once it is added and saved in the registry's settings.
{{% /alert %}}
8. Click on the **Create registry** button. 

{{< img src="/images/registry/create_registry.gif" alt="" >}}

For example, the preceding image shows a custom registry called "Fine_Tuned_Models" that a user is about to create. The registry is set to **Restricted** which means that only members that are manually added to the "Fine_Tuned_Models" registry will have access to this registry.