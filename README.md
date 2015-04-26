# Plugly

Plugly is a framework to alter behavior of existing .NET classes without a direct inheritance and method overrides.
The framework is built on top of [Castle.Core](https://github.com/castleproject/Core) library and is using *DynamicProxy* interceptors to extend the methods behavior and mixins to implement additional interfaces and add extra properties to existing classes.

The integrations to the most common Inversion-of-Control containers are supported out of the box.

## Installation

Just install the corresponding NuGet package:

	PM> Install-Package Plugly

All assemblies are signed.

## Basic Usage

Lets imagine that there is some class representing a customer:

```cs
public class Customer
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    
    public virtual string GetFullName()
    {
        return LastName + ", " + FirstName;
    }
}
```

Notice, that the class is **not sealed** and `GetFullName` method is **virtual**, otherwise it could not be customized.

Now, we create an extension which alters the behavior of the `GetFullName` method:

```cs
Customizer customizer = GetCustomizer(); // this method must return a singleton Customizer instance
customizer.Setup<Customer>()
    .Override<string>(c => c.GetFullName(), c => c.FirstName + " " + c.LastName)
    ;
```

After that, the customer instances should be created in the next way:

```cs
Customizer customizer = GetCustomizer(); // this method must return a singleton Customizer instance
var customer = customizer.CreateInstance<Customer>();
customer.FirstName = "John";
customer.LastName = "Doe";
Debug.Assert(customer.GetFullName() == "John Doe");
```

## Advanced extension with a mixin

Below is an advanced customization scenario example which also demonstrates the support of mixins and object initializers.
The following code extends the existing `Customer` class by appending a `MiddleName` property and including it in the output of the `GetFullName` method. Additionally, the following example demonstrates the possibility to add extra initialization of new class instances.

```cs
public interface IMyExtension
{
    string MiddleName { get; set; }
}

class MyExtension : IMyExtension
{
    public string MiddleName { get; set; }
    
    [Customization] // customization methods must be static, accept the target object as a first argument and be marked with a CustomizationAttribute
    static void __init(Customer customer) // '__init' is a special name for object initializer
    {
        customer.FirstName = "noname";
    }
    
    [Customization]
    static string GetFullName(Customer customer)
    {
        var ext = (IMyExtension)customer;
        var fullName = customer.GetFullName(); // invokes the inner customization or an original method
        if (ext.MiddleName != null)
            fullName = fullName.Replace(" ", " " + ext.MiddleName + " ");
        return fullName;
    }
}
```

Then the above extension is plugged-in this way:

```cs
GetCustomizer().Setup<Customer>()
    .Override<string>(c => c.GetFullName(), c => c.FirstName + " " + c.LastName)
    .ExtendWith<MyExtension>()
    .InitializeWith(c => c.LastName = "unknown");
```

And here is the usage example:

```cs
var customer = GetCustomizer().CreateInstance<Customer>();
dynamic dynamicCustomer = customer;
dynamicCustomer.MiddleName = "midname";
Debug.Assert(customer.GetFullName() == "noname midname unknown");
```

Notice, that:
1. Both initializers are invoked in the order they have been registered (both the `FirstName` and `LastName` properties are initialized with default values).
2. Both customizations for `GetFullName` method are applied in the **reverse order** as they have been registered.

For all available features, possibilities and usage examples take a look at the unit-tests project.

## Integration

### Unity Container

Unity Container integration is enabled by:

1\. Install the corresponding NuGet package

	PM> Install-Package Plugly.Unity
    
2\. Add the Plugly Unity extension
```cs
var container = new UnityContainer();
container.AddNewExtension<Plugly.Unity.Extension>();
```
3\. Use the customizer
```cs
var customizer = container.Resolve<Customizer>();
customizer.SetDefaultBuildUp(true) // enables dependency injection on all customized classes,
    .Setup<Customer>().BuildUp(false); // except the Customer class
```
## Performance

The performance impact is very low due to staticly typed customizations. The exact numbers can be obtained by running the **Plugly.Performance** console application included in the solution.

## Copyright

Copyright � 2015 Dmitry Moskalyk

## License

The project is licensed under the [MIT license](https://github.com/lokiworld/Plugly/blob/master/LICENSE.md).