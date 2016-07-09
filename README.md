# WebAPI

```cs

```

```cs
//Getting an Employee the WCF Way
[ServiceContract]
public interface IEmployeeService
{
	[OperationContract]
	[WebGet(UriTemplate = "/Employees/{id}")]
	Employee GetEmployee(string id);
}
	
public class EmployeeService : IEmployeeService
{
	public Employee GetEmployee(string id)
	{
		return new Employee() { Id = id, Name = "John Q Human" };
	}
}

[DataContract]
public class Employee
{
	[DataMember]
	public int Id { get; set; }
	
	[DataMember]
	public string Name { get; set; }
	
	// other members
}


```


```cs
//Getting an Employee the ASP.NET Web API Way/
public class EmployeeController : ApiController
{
	public Employee Get(string id)
	{
		return new Employee() { Id = id, Name = "John Q Human" };
	}
}
```



```cs
public class EmployeesController : ApiController
{
	private static IList<Employee> list = new List<Employee>()
	{
		new Employee()
		{
		Id = 12345, FirstName = "John", LastName = "Human"
		},
		new Employee()
		{
		Id = 12346, FirstName = "Jane", LastName = "Public"
		},
		new Employee()
		{
		Id = 12347, FirstName = "Joseph", LastName = "Law"
		}
	};
	
	// GET api/employees
	public IEnumerable<Employee> Get()
	{
		return list;
	}
	
	// GET api/employees/12345
	public Employee Get(int id)
	{
		return list.First(e => e.Id == id);
	}

	// POST api/employees
	public void Post(Employee employee)
	{
		int maxId = list.Max(e => e.Id);
		employee.Id = maxId + 1;
		list.Add(employee);
	}
	
	// PUT api/employees/12345
	public void Put(int id, Employee employee)
	{
		int index = list.ToList().FindIndex(e => e.Id == id);
		list[index] = employee;
	}
	
	// DELETE api/employees/12345
	public void Delete(int id)
	{
		Employee employee = Get(id);
		list.Remove(employee);
	}
}
```

## Choosing Configuration over Convention
```cs
public class EmployeesController : ApiController
{
	private static IList<Employee> list = new List<Employee>()
	{
		new Employee()
		{
			Id = 12345, FirstName = "John", LastName = "Human"
		},
		new Employee()
		{
			Id = 12346, FirstName = "Jane", LastName = "Public"
		},
		new Employee()
		{
			Id = 12347, FirstName = "Joseph", LastName = "Law"
		}
	};
	// Following action methods are commented out
	
	[AcceptVerbs("GET")]
	public Employee RetrieveEmployeeById(int id)
	{
		return list.First(e => e.Id == id);
	}
	
	[HttpGet]
	public Employee RetrieveEmployeeById(int id)
	{
		return list.First(e => e.Id == id);
	}
}

```
## WebApiConfig Class with RPC-style Mapping Removed
```cs
public static class WebApiConfig
{
	public static void Register(HttpConfiguration config)
	{
		config.Routes.MapHttpRoute(
			name: "RpcApi",
			routeTemplate: "api/{controller}/{action}/{id}",
			defaults: new { id = RouteParameter.Optional }
		);
		
		config.Routes.MapHttpRoute(
			name: "DefaultApi",
			routeTemplate: "api/{controller}/{id}",
			defaults: new { id = RouteParameter.Optional }
		);
	}
}
```


## Routing
```cs
public static class WebApiConfig
{
    public static void Register(HttpConfiguration config)
    {
		config.Routes.MapHttpRoute(
			name: "DefaultApi",
			routeTemplate: "api/{controller}/{id}",
			defaults: new { id = RouteParameter.Optional });
 
		config.Routes.MapHttpRoute( 
			name: "PostByDate", 
			routeTemplate: "api/Posts/{year}/{month}/{day}", 
			defaults: new { controller = "Posts", month = RouteParameter.Optional, day = RouteParameter.Optional } );
 
		config.Routes.MapHttpRoute( 
			name: "PostByDate", 
			routeTemplate: "api/{controller}/{year}/{month}/{day}", 
			defaults: new { month = RouteParameter.Optional, day = RouteParameter.Optional } ); 
		
		config.Routes.MapHttpRoute( 
			name: "PostByDate", 
			routeTemplate: "api/Posts/{year}/{month}/{day}", 
			defaults: new { 
				controller = "Posts", month = RouteParameter.Optional, day = RouteParameter.Optional 
				}, 
			constraints: new { 
				month = @"\d{0,2}", day = @"\d{0,2}" } 
				);
		
		///api/Posts/Category/10
		config.Routes.MapHttpRoute( 
			name: "PostsCustomAction",
			routeTemplate: "api/{controller}/{action}/{id}", 
			defaults: new { id = RouteParameter.Optional } );
		

        config.Routes.MapHttpRoute("product-get", "products/{productId}",
            new { controller = "Products", action = "GetProduct", logging = false },
            new { httpMethod = new HttpMethodConstraint("GET") });
        
        config.Routes.MapHttpRoute("product-list", "products",
            new { controller = "Products", action = "GetProducts", logging = false },
            new { httpMethod = new HttpMethodConstraint("GET") });
        
        config.Routes.MapHttpRoute("product-create", "products",
            new { controller = "Products", action = "PostProduct", logging = true },
            new { httpMethod = new HttpMethodConstraint("POST") });
        
        config.Routes.MapHttpRoute("product-update", "products/{productId}",
            new { controller = "Products", action = "UpdateProduct", logging = true },
            new { httpMethod = new HttpMethodConstraint("PUT") });
        
        config.Routes.MapHttpRoute("product-delete", "products/{productId}",
            new { controller = "Products", action = "DeleteProduct", logging = true },
            new { httpMethod = new HttpMethodConstraint("DELETE") });
    }
}
```

## Adding a constraint
```cs
//Works: http://localhost:55778/api/123/employees/12345
//Doesn't Works: http://localhost:55778/api/abc/employees/12345
//by adding a new {orgid} variable and adding a constraint, we have made sure the URI must include a new URI segment immediately after api and that it must be a number.
config.Routes.MapHttpRoute(
	name: "DefaultApi",
	routeTemplate: "api/{orgid}/{controller}/{id}",
	defaults: new { id = RouteParameter.Optional },
	constraints: new { orgid = @"\d+" }
);
```


## Retrieval of Employees by Department
```cs
//http://localhost:55778/api/employees?department=2
public IEnumerable<Employee> GetByDepartment(int department)
{
	int[] validDepartments = {1, 2, 3, 5, 8, 13};
	if (!validDepartments.Any(d => d == department))
	{
		var response = new HttpResponseMessage()
		{
			StatusCode = (HttpStatusCode)422, // Unprocessable Entity
			ReasonPhrase = "Invalid Department"
		};
		throw new HttpResponseException(response);
	}
	return list.Where(e => e.Department == department);
}
```



## It is possible to apply multiple conditions based on parameters. 
```cs
http://localhost:port/api/employees?department=2&lastname=Smith


Listing 1-21. Retrieving an Employee by Applying Two Conditions
public IEnumerable<Employee> Get([FromUri]Filter filter)
{
	return list.Where(e => e.Department == filter.Department &&
	e.LastName == filter.LastName);
}

//Create a class named Filter, under the Models folder.
public class Filter
{
	public int Department { get; set; }
	public string LastName { get; set; }
}
```


## Creating a Resource with a Server-Generated Identifier
```cs
//Creating an Employee using HTTP POST
public HttpResponseMessage Post(Employee employee)
{
	int maxId = list.Max(e => e.Id);
	employee.Id = maxId + 1;
	list.Add(employee);
	
	var response = Request.CreateResponse<Employee>(HttpStatusCode.Created, employee);
	
	string uri = Url.Link("DefaultApi", new { id = employee.Id });
	response.Headers.Location = new Uri(uri);
	
	return response;
}

//Output: Response Status Code and Headers
HTTP/1.1 201 Created
Date: Mon, 26 Mar 2013 07:35:07 GMT
Location: http://localhost:55778/api/employees/12348
Content-Type: application/json; charset=utf-8
```


## Creating a Resource with a Client-Supplied Identifier

//A resource such as an employee can be created by an HTTP PUT to the URI http://localhost:port/api/ employees/12348, 
//where the employee with an ID of 12348 does not exist until this PUT request is processed.
//In this case, the request body contains the JSON/XML representation of the resource being added, which is the new employee.

```cs
Creating an Employee using HTTP PUT
public HttpResponseMessage Put(int id, Employee employee)
{
	if (!list.Any(e => e.Id == id))
	{
		list.Add(employee);
		var response = Request.CreateResponse<Employee>(HttpStatusCode.Created, employee);
		
		string uri = Url.Link("DefaultApi", new { id = employee.Id });
		response.Headers.Location = new Uri(uri);
		return response;
	}
	return Request.CreateResponse(HttpStatusCode.NoContent);
}
```

## Overwriting a Resource
//A resource can be overwritten using HTTP PUT. This operation is generally considered the same as updating the resource, but there is a difference.
```cs
public HttpResponseMessage Put(int id, Employee employee)
{
	int index = list.ToList().FindIndex(e => e.Id == id);
	if (index >= 0)
	{
		list[index] = employee; // overwrite the existing resource
		return Request.CreateResponse(HttpStatusCode.NoContent);
	}
	else
	{
		list.Add(employee);
		var response = Request.CreateResponse<Employee>
		(HttpStatusCode.Created, employee);
		string uri = Url.Link("DefaultApi", new { id = employee.Id });
		response.Headers.Location = new Uri(uri);
		return response;
	}
}
```

## Updating a Resource
//A resource such as an employee in our example can be updated by an HTTP POST to the URI
//http://server/api/employees/12345, where the employee with an ID of 12345 already exists in the system.
```cs
public HttpResponseMessage Post(int id, Employee employee)
{
	int index = list.ToList().FindIndex(e => e.Id == id);
	if (index >= 0)
	{
		list[index] = employee;
		return Request.CreateResponse(HttpStatusCode.NoContent);
	}
	return Request.CreateResponse(HttpStatusCode.NotFound);
}
```

## Deleting a Resource
```cs
public void Delete(int id)
{
	Employee employee = Get(id);
	list.Remove(employee);
}
```
