# Developing New Provider

This section will go through what is needed to develop you own provider for cloudquery and optionally publish it in on cloudquery hub.

Before continuing it is recommended to get familiar with [cloudquery architecture](architecture.md).

Cloudquery providers are utilising cq-provider-sdk which abstracts most of the TL \(in ETL, extract-transform-load\). So as a developer you will only have to implement the \("E" in "ETL"\) initialising, authentication and fetching the data via the 3rd party APIs while the SDK will take care of transforming the data and loading into the database. Also your provider will get support out of-of-the-box for new features and things like other database support as cloudquery-core progress. 

Here is a template project from which you can create your own [https://github.com/cloudquery/cq-provider-template](https://github.com/cloudquery/cq-provider-template).

We will go through the files in the template and explain each part that you need to implement.

#### **resources/provider.go**

```go
func Plugin() *plugin.Provider {
	return &plugin.Provider{
		Name:      "your_provider_name",
		Configure: provider.Configure,
		ResourceMap: map[string]*schema.Table{
			"resource_name": resources.ResourceName(),
		},
		Config: func() provider.Config {
			return &provider.Config{}
		},
	}

}
```

In this file everything is already set-up for you and you only need to change `Name` to match your provider name and add new resource to `ResourceMap`  as you implement them and add them to your provider.

**client/config.go**

```go
package client

// Provider Configuration
type Account struct {
    Name string `hcl:"name,optional"`
}

type Config struct {
    Account []Account `hcl:"account,block"`
    User string `hcl:"user,optional"`
    Debug bool `hcl:"debug,optional"`
}


// Pass example to cloudquery when cloudqueyr init <provider> will be called
func (c Config) Example() string {
    return `configuration {
    
	// Optional. create multiple blocks of accounts the provider will run
  // account {
	// name = <Name attribute>
	// }
	
	// Optional. Some field we decided to add
	user = "cloudquery"
	// Optional. Enable Provider SDK debug logging.
   debug = false  
}
`
}
```

Here you define the "hcl block" configuration that the user can pass to your provider. This config is parsed and populated by the SDK so you don’t need to deal with HCL marshaling/unmarshalling.  The populated config object is passed to **provider.Configure** function in **client.go.**

**provider/client.go**

```go
type Client struct {
	// This is a client that you need to create and initialize in Configure
	// It will be passed for each resource fetcher.
	logger     hclog.Logger
}

func (c *Client) Logger() hclog.Logger {
	return c.logger
}


func Configure(logger hclog.Logger, providerConfig interface{}) (schema.ClientMeta, error) {
	providerConfig := config.(*Config)

	// Init your client and 3rd party clients using the user's configuration
	// passed by the SDK
	client := Client{
		logger:   logger,
	}

	// Return the initialized client and it will be passed to your resources
	return &client, nil
}
```

This function is called before fetching any resources. The provider has a chance to read the top-level configuration, init and authenticate all needed 3rd party clients and return your initialised object that will be passed to each one of your fetchers. 

**resources/demo\_resources.go**

In this directory you will create a new file for each resource. Each resource may contain one or more related tables. See documentation inline.

```go
package resources

import (
	"context"
	"github.com/cloudquery/cq-provider-sdk/plugin/schema"
	"github.com/cloudquery/cq-provider-template/client"
)

func DemoResource() *schema.Table {
	return &schema.Table{
		// Required. Table Name
		Name:         "provider_demo_resources",
		// Required. Fetch data for table. See fetchDemoResources
		Resolver:     fetchDemoResources,
		// Optional. DeleteFilter returns a list of key/value pairs to add when truncating this table's data from the database.
		// DeleteFilter: nil, // func(meta ClientMeta) []interface{}

		// Optional. Returns re-purposed meta clients. The sdk will execute the table with each of them. useful if you want to execute for different accounts, etc...
		// Multiplex:    nil, // func(meta ClientMeta) []ClientMeta

		// Optional. checks if returned error from table resolver should be ignored. If it return true the sdk will ignore and continue instead of aborting.
		// IgnoreError:  nil, // IgnoreErrorFunc func(err error) bool

		// Optional. Post resource resolver is called after all columns have been resolved, and before resource is inserted to database.
		// PostResourceResolver: nil, // RowResolver func(ctx context.Context, meta ClientMeta, resource *Resource) error


		Columns: []schema.Column{
			{
				Name:     "account_id",
				Type:     schema.TypeString,
				// Optional. You can have a special column resolver if the column name doesnt match the name or it's just an additional
				//  column that needs to get the data from somewhere else
				Resolver: customColumnResolver,
			},
			{
				Name:     "region",
				Type:     schema.TypeString,
			},
			{
				Name: "creation_date",
				Type: schema.TypeTimestamp,
			},
			{
				Name: "name",
				Type: schema.TypeString,
				// schema.PathResolver is a utility function that get the data from a different name in the struct
				// Resolver: schema.PathResolver("other_name_in_struct"),
			},
		},
		// A table can have relations
		//Relations: []*schema.Table{
		//	{
		//		Name:     "provider_demo_resource_children",
		//		Resolver: fetchDemoResourceChildren,
		//		Columns: []schema.Column{
		//			{
		//				Name:     "bucket_id",
		//				Type:     schema.TypeUUID,
		//				Resolver: schema.ParentIdResolver,
		//			},
		//			{
		//				Name:     "resource_id",
		//				Type:     schema.TypeString,
		//				Resolver: schema.PathResolver("Grantee.ID"),
		//			},
		//			{
		//				Name:     "type",
		//				Type:     schema.TypeString,
		//				Resolver: schema.PathResolver("Grantee.Type"),
		//			},
		//		},
		//	},
		//},
	}
}


// ====================================================================================================================
//                                               Table Resolver Functions
// ====================================================================================================================
func fetchDemoResources(ctx context.Context, meta schema.ClientMeta, parent *schema.Resource, res chan interface{}) error {
	c := meta.(*client.Client)
	_ = c
	// Fetch using the third party client and put the result in res
	// res <- c.ThirdPartyClient.getDat()
	return nil
}

func customColumnResolver(ctx context.Context, meta schema.ClientMeta, resource *schema.Resource, c schema.Column) error {
	resource.Set("column_name", "value")
	return nil
}
```

Essentially for each resource that you support you just need to define two things

* The schema - how the table will look in the database - column names and types. 
* Implement the main table resolver function that will fetch the data from the 3rd party SDK and pass it to the SDK. 
  * The SDK will automatically read the data and insert into the table column using a default naming convention. The default naming convention is to CamelCase i.e if a column-name is `some_name` the field name in the struct that you pass to the sdk should be: `SomeName` . If you want a different name or logic you can implement a column resolver.
