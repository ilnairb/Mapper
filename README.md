# Mapper

**Welcome!**  

A lightweight object-to-object mapper for .NET, supporting deep mapping, type conversion, nested objects, and collections. Designed for high performance, flexibility, and clean code, making it easy to map between different data models.

This repository is for discussion, issue tracking, and user support related to the Mapper library.

> **Note:**  
> The source code for Mapper is not available in this repository. Please use this space for reporting issues, requesting features, or community discussions.

## How to Use This Repository

- Report bugs or issues
- Request new features
- Ask questions or discuss best practices

## Source Code

The source code for Mapper is not published here.  
For more details or contribution opportunities, please contact the maintainer.

## Need Help?

- [Open an issue](https://github.com/ilnairb/Mapper/issues)
- [Start a discussion](https://github.com/ilnairb/Mapper/discussions)

Thank you for your interest and support!

# Mapper Quick Start Guide

## 1. Installation

Install this nuget package to your application.

---

## 2. Basic Usage: Mapping Between Classes

Suppose you have these two classes:

```csharp
public class Person
{
    public int Id { get; set; }
    public string Name { get; set; }
    public Address Address { get; set; }
}

public class PersonDto
{
    public int Id { get; set; }
    public string Name { get; set; }
    public AddressDto Address { get; set; }
}
```

Create a mapper instance:

```csharp
var mapper = new Mapper();
```

Map an object

```csharp
Person person = GetPersonFromDb();
PersonDto dto = mapper.Map<Person, PersonDto>(person);
```

- All matching properties (by name and compatible type) will be mapped automatically.
- Nested objects and collections are mapped recursively.

## 3. Mapping Collections

```csharp
List<Person> people = GetPeople();
List<PersonDto> dtos = mapper.Map<List<Person>, List<PersonDto>>(people);
```

## 4. Ignore Properties (Optional)

You can prevent specific properties from being mapped by using the [IgnoreMap] attribute:

```csharp
public class PersonDto
{
    public int Id { get; set; }

    [IgnoreMap]
    public string Name { get; set; }
}
```
Properties marked with [IgnoreMap] will not be copied during mapping.

Alternatively, you can configure the mapper globally to ignore certain properties based on the name (case insensitive):

```csharp
_mapper = new Mapper();

_mapper.IgnoreProperty("Test");
_mapper.IgnorePropertiesStartingWith("_");
_mapper.AddIgnorePropertyRule(n => n.EndsWith("test"));
```

## 5. Projection for Queryables (ProjectTo)

For scenarios like Entity Framework where you want expression-based projection:

```csharp
IQueryable<Person> people = dbContext.People;
var projected = people.ProjectTo<PersonDto>(mapper);
```

This creates an IQueryable<PersonDto> that the database provider can optimize. IOrderedQueryable is supported as well.

## 6. Diagnostics (Optional)

After mapping, you can access mapping diagnostics and warnings:

```csharp
var diagnostics = mapper.Diagnostics;
foreach (var warning in diagnostics.Warnings)
{
    Console.WriteLine(warning);
}
```

## 7. Auto Mapping Configuration

While Mapper can perform mapping on the fly by inspecting source and destination classes at runtime, it is **recommended to configure mappings at application startup**. This way, mappings can be registered and cached for greater performance and reliability.

### Create a Marker Interface 

```csharp
public interface IMapFrom<T>
{
    public void Mapping(IMapper mapper);
}
```
### Implement This Interface in your DTO Classes

```csharp
public class PersonDto : IMapFrom<Person>
{
    public int Id { get; init; }
    public string Name { get; set; }

    // This method configures the map for this DTO
    public void Mapping(IMapper mapper)
    {
        mapper.CreateMap<Person, PersonDto>();
        mapper.CreateMap<PersonDto, Person>();
    }
}
```

### Implement IProfile to Register All Mappings 

```csharp
public class MappingProfile : IProfile
{
	public void Configure(IMapper mapper)
	{
		ApplyIMapFromMappings(mapper, Assembly.GetExecutingAssembly());
	}

	private void ApplyIMapFromMappings(IMapper mapper, Assembly assembly)
	{
		var mapFromType = typeof(IMapFrom<>);
		var mappingMethodName = nameof(IMapFrom<object>.Mapping);
		var argumentTypes = new[] { typeof(IMapper) };

		bool HasMapFromInterface(Type t) =>
			t.IsGenericType && t.GetGenericTypeDefinition() == mapFromType;

		var types = assembly.GetExportedTypes()
			.Where(t => t.GetInterfaces().Any(HasMapFromInterface))
			.ToList();

		foreach (var type in types)
		{
			var instance = Activator.CreateInstance(type);
			if (instance == null) continue;

			// Try method directly on the class
			var methodInfo = type.GetMethod(mappingMethodName, argumentTypes);
			if (methodInfo != null)
			{
				methodInfo.Invoke(instance, new object[] { mapper });
				continue;
			}

			// Fallback to interface method (explicit interface implementation)
			foreach (var @interface in type.GetInterfaces().Where(HasMapFromInterface))
			{
				var interfaceMethod = @interface.GetMethod(mappingMethodName, argumentTypes);
				if (interfaceMethod != null)
				{
					interfaceMethod.Invoke(instance, new object[] { mapper });
				}
			}
		}
	}
}
```

### Finally register the mapper:

```csharp
services.AddMapper(Assembly.GetExecutingAssembly());
```

The mapper will automatically scan the assembly, find all implementations of IProfile, and run their Configure method.
By doing this, **all mappings are registered and cached at startup, ensuring efficient mapping during runtime**.

**Tip**:
You can implement multiple IProfile classes (e.g., per module or feature) and register them all in this way for clean and scalable configuration. There is no profile level configuration. All Profiles are treated in the same way.

## 8. Notes & Best Practices

- By default, all matching properties are mapped.
- Use init-only properties in DTOs to make values immutable after mapping, if desired.
- Keep mapping logic simple and free of business/presentation rules for best maintainability.
- In DTO classes, avoid using bi-directional relationship
