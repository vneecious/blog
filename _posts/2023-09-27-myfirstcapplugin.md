---
title: "SAP CAP SDM Plugin"
date: 2023-09-27
---

I have just released the initial version of [`sap-cap-sdm-plugin`](https://github.com/vneecious/sap-cap-sdm-plugin), a plugin designed to streamline the integration between CAP and DMS. The inspiration for this plugin arose after reading Daniel's [blog](https://blogs.sap.com/2023/04/30/reusable-components-for-cap-with-cds-plugin/) about `cds.plugins`. In his example, Daniel creates a custom annotation that appends an emoji to the end of the text for properties that are annotated with it. Although simplistic, his example sparked some ideas in me.

Recently, I finished developing [`sap-cloud-cmis-client`](https://github.com/vneecious/sap-cloud-cmis-client), another library aimed at simplifying the integration of cloud projects (cap-js or @sap-cloud-sdk) on Cloud Foundry with DMS. Despite its utility, the library did not fully meet my expectations, as it still required considerable effort from developers. This contradicted my goal of achieving seamless integration.

Daniel's example then led me to ask a pivotal question: why not integrate DMS into CAP projects through entity annotations? Developers could simply use the `@Sdm.Entity` annotation to designate an SDM entity, then map its properties to specific CMIS properties using `@Sdm.Field`, all without needing to deal with additional concerns. There would be no need to worry about CMIS, token propagation, or anything else, as the plugin and `sap-cloud-cmis-client` would handle everything behind the scenes.

After addressing numerous questions, reviewing extensive documentation, and investing some spare hours, the result emerged exactly as I envisioned.

## How to Use

First, install the plugin:

```sh
npm i sap-cap-sdm-plugin
```

Then, include it in the `cds.requires` of your project:

```package.json
"cds": {
    "requires": {
        "sap-cap-sdm-plugin": {
            "impl": "sap-cap-sdm-plugin",
            "settings": {
                "destination": "<YOUR_SDM_DESTINATION_NAME>",
                "repositoryId": "<YOUR_REPOSITORY_ID>" // Optional. Remove if you have only one repository.
            }
        }
    }
},
```

Currently, only two settings are available: 
 1. `destination` (mandatory): where the user must specify the destination related to the DMS service.
 2. `repositoryId` (optional): where the user can identify the working repository ID.

You can also set these for different profiles:

```package.json
"cds": {
    "requires": {
        "sap-cap-sdm-plugin": {
            "impl": "sap-cap-sdm-plugin",
            "[development]": { // development profile
              "settings": {
                  "destination": "<YOUR_SDM_DESTINATION_NAME>",
                  "repositoryId": "123"
              }
            },
            "[production]": { // production profile
              "settings": {
                  "destination": "<YOUR_SDM_DESTINATION_NAME>",
                  "repositoryId": "456"
              }
            }
        }
    }
}
```

With that done, create an entity linked with the DMS, using the appropriate annotations. Here’s an example:

```cds
service SampleService {
    @cds.persistence.skip
    @Sdm.Entity
    entity Files {
        key id       : String      @Sdm.Field      : { type : 'property', path : 'cmis:objectId' };
        name         : String      @Sdm.Field      : { type : 'property', path : 'cmis:name' };
        content      : LargeBinary @Core.MediaType : contentType  @Core.ContentDisposition.Filename : name;
        contentType  : String      @Core.IsMediaType
                                   @Sdm.Field      : { type : 'property', path : 'cmis:contentStreamMimeType' };
        createdBy    : String      @Sdm.Field      : { type : 'property', path : 'cmis:createdBy' };
        creationDate : Date        @Sdm.Field      : { type : 'property', path : 'cmis:creationDate' };
    }
}
```

The `@Sdm.Entity` annotation indicates that the entity will be linked with the DMS. This annotation is crucial; without it, subsequent annotations won't have any effect. 
The `@Sdm.Field` annotation denotes field-level relationships. Currently, only two types are available: 

1. `property`, which associates an entity property with a CMIS property, like `Files.name` with `cmis:name`, and
2. `url`, which returns the direct URL to the document. This type optionally accepts a `path`, which should be the desired streamId, especially useful for retrieving URLs, like thumbnails (`path: 'sap:thumbnailContentStreamId').

The entity is annotated with `@cds.persistence.skip` as there’s no need to create it in the database.

Moreover, it's possible to establish navigation as well. For instance, if a Purchase Order must have multiple attachments, just create the Attachments entity as `@Sdm.Entity` and establish associations as you would with a "normal" entity. The only difference here is that `$expand` won’t work since the table doesn’t exist in the database. So, this should be resolved through *navigation*, not *expansion*. In other words, you should use `PurchaseOrder('123')/attachments` instead of `PurchaseOrder('123')?$expand=attachments`. Alternatively, you can implement a custom handler to deal with `$expand`, as [described here](https://cap.cloud.sap/docs/guides/using-services#handle-navigations-across-local-and-remote-entities).

The plugin does have its limitations. For now, it doesn’t support versioned repositories, thumbnail generation, extensions, working with multiple repositories, dealing with directories, and a few other things. However, for 90% of cases, where the only requirement is attaching documents to a business process, it works very well.

I hope it proves as useful to you as it has been to me. And if you run into any issues, [why not help me solve them](https://github.com/vneecious/sap-cap-sdm-plugin/issues)? 🤓
