# WebAPI

# TOC
* [Getting an Employee the WCF Way](#getting-an-employee-the-wcf-way)
* [Getting an Employee the ASP.NET Web API Way](#getting-an-employee-the-aspnet-web-api-way)
* [Simple REST API](#simple-rest-api)
* [Routing](#routing)
* [Controllers](#controllers )
* [Actions](#actions)
* [Message Handlers](#message-handlers)
* [Filters](#filters)
* [Media Type Formatters ](#media-type-formatters )
* [Model Binding](#model-binding)
* [Input Validation](#input-validation)
* [Dependency Resolution](#dependency-resolution)
* [Testing](#testing)
* [Securing the Service](securing-the-service)
* [Optimization and Performance](#optimization-and-performance)
* [Hosting](#hosting)
* [Error Handling](#error-handling)
* [Tracing and Logging](#Tracing-and-Logging)
* [API Documentation](#api-documentation)




## Getting an Employee the WCF Way
```cs
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

## Getting an Employee the ASP.NET Web API Way
```cs
public class EmployeeController : ApiController
{
	public Employee Get(string id)
	{
		return new Employee() { Id = id, Name = "John Q Human" };
	}
}
```

## Simple REST API 

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


## Routing
```cs
// WebApiConfig.cs
public static class WebApiConfig
{
	public static void Register(HttpConfiguration config)
	{
	
		// Default catch-all
		config.Routes.MapHttpRoute(
			name: "DefaultApi",
			routeTemplate: "api/{controller}/{id}",
			defaults: new { id = RouteParameter.Optional }
		);
		
		//RPC-style
		config.Routes.MapHttpRoute(
            		name: "RpcApi",
            		routeTemplate: "api/{controller}/{action}/{id}",
            		defaults: new { id = RouteParameter.Optional }
        	);
        
	    // Matches route with the taskNum parameter
		config.Routes.MapHttpRoute(
			name: "FindByTaskNumberRoute",
			routeTemplate: "api/{controller}/{taskNum}",
			defaults: new { taskNum = RouteParameter.Optional }
		);
	
		config.Routes.MapHttpRoute( 
			name: "PostByDate", 
			routeTemplate: "api/Posts/{year}/{month}/{day}", 
			defaults: new { controller = "Posts", month = RouteParameter.Optional, day = RouteParameter.Optional } );
		
		//Works: http://localhost:55778/api/123/employees/12345
		//Doesn't Works: http://localhost:55778/api/abc/employees/12345
		config.Routes.MapHttpRoute(
			name: "DefaultApi",
			routeTemplate: "api/{orgid}/{controller}/{id}",
			defaults: new { id = RouteParameter.Optional },
			constraints: new { orgid = @"\d+" }
		);

		config.Routes.MapHttpRoute( 
			name: "PostByDate", 
			routeTemplate: "api/Posts/{year}/{month}/{day}", 
			defaults: new { 
				controller = "Posts", month = RouteParameter.Optional, day = RouteParameter.Optional}, 
			constraints: new { 
				month = @"\d{0,2}", day = @"\d{0,2}" } 
				);
				
		//Using a Built-in Constraint
		//Add - using System.Web.Http.Routing.Constraints;
		config.Routes.MapHttpRoute(
			name: "ChromeRoute",
			routeTemplate: "api/today/{action}",
			defaults: new { controller = "today" },
			constraints: new {
				useragent = new UserAgentConstraint("Chrome"),
				action = new RegexRouteConstraint("daynumber|othermethod")
			}
		);		
		
		//api/Posts/Category/10
		config.Routes.MapHttpRoute( 
			name: "PostsCustomAction",
			routeTemplate: "api/{controller}/{action}/{id}", 
			defaults: new { id = RouteParameter.Optional } );
		

		config.Routes.MapHttpRoute(
            name: "DefaultApi",
            routeTemplate: "api/{controller}/{id}",
            defaults: new { id = RouteParameter.Optional },
            constraints: null,
            handler: new CustomHttpControllerDispatcher(config)
        );

		//Defining a Route with Data Tokens
		config.Routes.Add(
			"CustomHandler",
			config.Routes.CreateRoute(
			routeTemplate: "api/{controller}/{action}",
			defaults: null,
			constraints: null,
			dataTokens: new Dictionary<string, object> {
			{ "response", "Tomorrow" }
			},
			handler: new CustomRouteHandler())
		);
	
        config.Routes.MapHttpRoute("product-delete", "products/{productId}",
            new { controller = "Products", action = "DeleteProduct", logging = true },
            new { httpMethod = new HttpMethodConstraint("DELETE") });
    }
}
```

### Route Registration in ASP.NET Web API with ASP.NET Hosting
```cs
protected void Application_Start(object sender, EventArgs e) {
	var config = GlobalConfiguration.Configuration;
	var routes = config.Routes;
	
	routes.MapHttpRoute(
		"DefaultHttpRoute",
		"api/{controller}/{id}",
		new { id = RouteParameter.Optional }
		);
	
	//Sample 2: Sample Route Registration	
	GlobalConfiguration.Configuration.Routes.MapHttpRoute(
		"DefaultHttpRoute",
		"api/{controller}/{id}",
		new { id = RouteParameter.Optional }
		);	
		
	//Having Multiple Web API Routes	
	var routes1 = GlobalConfiguration.Configuration.Routes;
		routes.MapHttpRoute(
		"DefaultHttpRoute",
		"api/{controller}/{id}",
		new { id = RouteParameter.Optional }
		);
		
	routes1.MapHttpRoute(
		"VehicleHttpRoute",
		"api/{vehicletype}/{controller}",
		defaults: new { },
		constraints: new { controller = "^vehicles$" }
		);
}

```


## Controllers


### Controller Sample

public class CarsController : ApiController {
	private readonly CarsContext _carsCtx = new CarsContext();
	
	//NonActionAttribute
	[NonAction]
	public string[] GetCars() {
		return new string[] {
		"Car 1",
		"Car 2",
		"Car 3"
		};
	}

	//Get Action
	public IEnumerable<Car> Get() {
		var cars = _carsCtx.All;
		
		HttpResponseMessage response =
		Request.CreateResponse<string[]>(HttpStatusCode.OK, cars);
		response.Headers.Add("X-Foo", "Bar");
		return response;
	}
	
	//Get Action
	public Car GetCar(int id) {
		var carTuple = _carsCtx.GetSingle(id);

		if (!carTuple.Item1) {
			var response = 	Request.CreateResponse(HttpStatusCode.NotFound);
			throw new HttpResponseException(response);
		}
		return carTuple.Item2;
	}


	//Post Action
	public HttpResponseMessage PostCar(Car car) {
		var createdCar = _carsCtx.Add(car);
		var response = Request.CreateResponse(HttpStatusCode.Created, createdCar);
		response.Headers.Location = new Uri(Url.Link("DefaultHttpRoute", new {	id = createdCar.Id }));
		return response;
	}
	
	//Put Action
	public Car PutCar(int id, Car car) {
		car.Id = id;
		if (!_carsCtx.TryUpdate(car)) {
			return new HttpResponseMessage(HttpStatusCode.NotFound);
		}
		return car;
	}
	
	//Delete Action Method
	public HttpResponseMessage DeleteCar(int id) {
		if (!_carsCtx.TryRemove(id)) {
			return new HttpResponseMessage(HttpStatusCode.NotFound);
		}
		return Request.CreateResponse(HttpStatusCode.OK);
	}
}


```cs
//Defining a Common Prefix
[RoutePrefix("api/today")]
public class TodayController : ApiController {
}

//Applying the Route Attribute to the Controller
[Route("{action=DayOfWeek}")]
public class TodayController : ApiController {
}

//Consolidating the Direct Routes
[Route("{action=DayOfWeek}/{day:range(0, 6)}")]
public class TodayController : ApiController {
}

//Using a Custom Route Attribute
[UserAgentConstraintRoute("{action=DayOfWeek}/{day:specval(2)}")]
public class TodayController : ApiController {
}

//Applying the CustomControllerConfigAttribute
[CustomControllerConfig]
public class CustomController : ApiController {
}
```



### A Sample Controller Action That Throws HttpResponseException
```cs
public class CarsController : ApiController {
	
	public string[] GetCars() {
		try {
			int left = 10,
			right = 0;
			var result = left / right;
		}
		catch (DivideByZeroException ex) {
			var faultedResponse = Request.CreateResponse(HttpStatusCode.InternalServerError, new HttpError(ex, includeErrorDetail: true));
			
			throw new HttpResponseException(faultedResponse);
		}
			return new[] {
			"Car 1",
			"Car 2",
			"Car 3"
			};
	}

}

```


### Retrieval of Employees by Department
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



### It is possible to apply multiple conditions based on parameters. 
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


### Creating a Resource with a Server-Generated Identifier
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


### Creating a Resource with a Client-Supplied Identifier

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

### Overwriting a Resource
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

### Updating a Resource
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

### Route
```cs
[RoutePrefixAttribute("api/employeeTasks")]
public class TasksController : ApiController
{
	[Route("{id:int:max(100)}")]
	public string GetTaskWithAMaxIdOf100(int id)
	{
		return "In the GetTaskWithAMaxIdOf100(int id) method, id = " + id;
	}
	
	[Route("{id:int:min(101)}")]
	[HttpGet]
	public string FindTaskWithAMinIdOf101(int id)
	{
		return "In the FindTaskWithAMinIdOf101(int id) method, id = " + id;
	}
{
```

## Actions
```cs
public class RsvpController : ApiController {
	[HttpGet]
	public IEnumerable<GuestResponse> Attendees() {
		return Repository.Responses.Where(x => x.WillAttend == true);
	}
	
	[HttpPost]
	public void Add(GuestResponse response) {
		if (ModelState.IsValid) {
		Repository.Add(response);
		}
	}
	
	//Binding Request Data As an Array
	[HttpGet]
	[HttpPost]
	public string SumNumbers([ModelBinder] int[] numbers) {
		return numbers.Sum().ToString();
	}
	
	//Binding Request Data As a Strongly Typed Collection
	[HttpGet]
	[HttpPost]
	public string SumNumbers([ModelBinder] List<int> numbers) {
		return numbers.Sum().ToString();
	}
	
	//Adding an Action Method
	[HttpGet]
	[Route("api/products/noop")]
	public IHttpActionResult NoOp() {
		return Ok();
	}
	
	//Using a StatusCodeResult
	public IHttpActionResult Delete(int id) {
		repo.DeleteProduct(id);
		return StatusCode(HttpStatusCode.NoContent);
	}
	
	//Using the ResponseMessage Method
	public IHttpActionResult Delete(int id) {
		repo.DeleteProduct(id);
		return ResponseMessage(new HttpResponseMessage(HttpStatusCode.NoContent));
	}
	
	//Using a Custom Action Method
	public IHttpActionResult Delete(int id) {
		repo.DeleteProduct(id);
		return new MyNoContentResult();
	}
	
	//Defining Direct Routes
	[HttpGet]
	[Route("api/today/dayofweek")]
	public string DayOfWeek() {
		return DateTime.Now.ToString("dddd");
	}

	//Defining Direct Routes
	[HttpGet]
	[Route("api/today/dayofweek/{day}")]
	public string DayOfWeek(int day) {
		return Enum.GetValues(typeof(DayOfWeek)).GetValue(day).ToString();
	}

	//To prevent the prefix from being applied
	[HttpGet]
	[Route("~/getdaynumber")]
	public int DayNumber() {
		return DateTime.Now.Day;
	}

	//Defining an Optional Segment
	[HttpGet]
	[Route("dayofweek/{day?}")]
	public string DayOfWeek(int day = -1) {	//set a default value on the action method parameter
		if (day != -1) {
			return Enum.GetValues(typeof(DayOfWeek)).GetValue(day).ToString();
		} else {
			return DateTime.Now.ToString("dddd");
		}
	}

	//Handling an Optional Segment
	[HttpGet]
	[Route("dayofweek/{day?}")]
	public IHttpActionResult DayOfWeek(int day = -1) {
		if (RequestContext.RouteData.Values.ContainsKey("day")) {
			return day != -1 	? Ok(Enum.GetValues(typeof(DayOfWeek)).GetValue(day).ToString()) 	: (IHttpActionResult)BadRequest("Value Out of Range");
		} else {
			return Ok(DateTime.Now.ToString("dddd"));
		}
	}

	//Defining a Default Segment Value
	[Route("dayofweek/{day=-1}")]
	public string DayOfWeek(int day) {
		if (day != -1) {
			return Enum.GetValues(typeof(DayOfWeek)).GetValue(day).ToString();
		} else {
			return DateTime.Now.ToString("dddd");
		}
	}

	//Applying a Constraint to a Direct Route
	[HttpGet]
	[Route("dayofweek/{day:int=-1}")]
	public string DayOfWeek(int day) {
		if (day != -1) {
			return Enum.GetValues(typeof(DayOfWeek)).GetValue(day).ToString();
		} else {
			return DateTime.Now.ToString("dddd");
		}
	}

	[HttpGet]
	[Route("dayofweek")]
	public string DayOfWeek() {
		return DateTime.Now.ToString("dddd");
	}

	[HttpGet]
	[Route("dayofweek/{day:range(0, 6)}")]
	public string DayOfWeek(int day) {
		return Enum.GetValues(typeof(DayOfWeek)).GetValue(day).ToString();
	}
	
	[AcceptVerbs("GET")]
	public Employee RetrieveEmployeeById(int id)
	{
		return list.First(e => e.Id == id);
	}

	//Applying the Order Property to the Route Attribute
	[HttpGet]
	[Route("~/getdaynumber", Order=1)]
	public int DayNumber() {
		return DateTime.Now.Day;
	}

	//Specifying HTTP Verbs
	[AcceptVerbs("GET", "HEAD")]
	public string DayOfWeek() {
		return DateTime.Now.ToString("dddd");
	}
	
	[CustomException]
	public Product Get(int id) {
		return products[id];
		//return products.Where(x => x.ProductID == id).FirstOrDefault();
	}
	
}		
```

## Securing the Service
```cs
//Applying Authorization in the ProductsController.cs File
[Authorize(Roles = "Administrators")]
public async Task DeleteProduct(int id) {
	await Repository.DeleteProductAsync(id);
}
```


### The Authorization Filter
```cs

[Route("", Name = "AddTaskRoute")]
[HttpPost]
[Authorize(Roles = Constants.RoleNames.Manager)]
public IHttpActionResult AddTask(HttpRequestMessage requestMessage, NewTask newTask)
{
	var task = _addTaskMaintenanceProcessor.AddTask(newTask);
	var result = new TaskCreatedActionResult(requestMessage, task);
	return result;
}
```

### A Message Handler to Support HTTP Basic Authentication
```cs

//IBasicSecurityService.cs
public interface IBasicSecurityService
{
	bool SetPrincipal(string username, string password);
}

//BasicSecurityService.cs
public class BasicSecurityService : IBasicSecurityService
{
	public virtual IPrincipal GetPrincipal(User user)
	{
		var identity = new GenericIdentity(user.Username, Constants.SchemeTypes.Basic);
		identity.AddClaim(new Claim(ClaimTypes.GivenName, user.Firstname));
		identity.AddClaim(new Claim(ClaimTypes.Surname, user.Lastname));
		var username = user.Username.ToLowerInvariant();
		
		switch (username)
		{
		case "bhogg":
			identity.AddClaim(new Claim(ClaimTypes.Role, Constants.RoleNames.Manager));
			identity.AddClaim(new Claim(ClaimTypes.Role, Constants.RoleNames.SeniorWorker));
			identity.AddClaim(new Claim(ClaimTypes.Role, Constants.RoleNames.JuniorWorker));
			break;
		case "jbob":
			identity.AddClaim(new Claim(ClaimTypes.Role, Constants.RoleNames.SeniorWorker));
			identity.AddClaim(new Claim(ClaimTypes.Role, Constants.RoleNames.JuniorWorker));
			break;
		case "jdoe":
			identity.AddClaim(new Claim(ClaimTypes.Role, Constants.RoleNames.JuniorWorker));
			break;
		default:
			return null;
		}
		return new ClaimsPrincipal(identity);
	}
}
```

### Auditing
```cs
public class UserAuditAttribute : ActionFilterAttribute
{
	private readonly ILog _log;
	private readonly IUserSession _userSession;
	public UserAuditAttribute()
	: this(WebContainerManager.Get<ILogManager>(), WebContainerManager.Get<IUserSession>())
	{
	}
	public UserAuditAttribute(ILogManager logManager, IUserSession userSession)
	{
		_userSession = userSession;
		_log = logManager.GetLog(typeof (UserAuditAttribute));
	}
	public override bool AllowMultiple
	{
		get { return false; }
	}
	
	public override Task OnActionExecutingAsync(HttpActionContext actionContext,
	CancellationToken cancellationToken)
	{
		_log.Debug("Starting execution...");
		var userName = _userSession.Username;
		return Task.Run(() => AuditCurrentUser(userName), cancellationToken);
	}
	
	public void AuditCurrentUser(string username)
	{
		// Simulate long auditing process
		_log.InfoFormat("Action being executed by user={0}", username);
		Thread.Sleep(3000);
	}
	public override void OnActionExecuted(HttpActionExecutedContext actionExecutedContext)
	{
		_log.InfoFormat("Action executed by user={0}", _userSession.Username);
	}
}
```


```cs
[HttpPost]
[UserAudit]
[Route("tasks/{taskId:long}/reactivations", Name = "ReactivateTaskRoute")]
public Task ReactivateTask(long taskId)
{
	var task = _reactivateTaskWorkflowProcessor.ReactivateTask(taskId);
	return task;
}
```

## Understanding Parameter and Model Binding

```cs
[HttpGet]
[HttpPost]
public int SumNumbers(int first, int second) {
	return first + second;
}

//Using a Model Class - Refer: Numbers.cs
[HttpGet]
[HttpPost]
public int SumNumbers(Numbers calc) {
	return calc.First + calc.Second;
}

//Adding an Action Method Parameter - refer: Operation.cs
[HttpGet]
[HttpPost]
public int SumNumbers(Numbers calc, Operation op) {
	int result = op.Add ? calc.First + calc.Second : calc.First - calc.Second;
	return op.Double ? result * 2 : result;
}

//Getting Values for a Complex Type from the Request URL
[HttpGet]
[HttpPost]
public int SumNumbers([FromUri] Numbers calc, [FromUri] Operation op) {
	int result = op.Add ? calc.First + calc.Second : calc.First - calc.Second;
	return op.Double ? result * 2 : result;
}

//Using the FromBody Attribute
[HttpGet]
[HttpPost]
public int SumNumbers([FromBody] int number) {
	return number * 2;
}


//Defining a Binding Rule in the WebApiConfig.cs File
config.ParameterBindingRules.Insert(0, typeof(Numbers), x => x.BindWithAttribute(new FromUriAttribute()));


//Numbers.cs
public class Numbers {
	public int First { get; set; }
	public int Second { get; set; }
}

//Operation.cs
public class Operation {
	public bool Add { get; set; }
	public bool Double { get; set; }
}
```

## Message Handlers
### HttpMessageHandler Abstract Base Class Implementation
```cs
namespace System.Net.Http
{
	public abstract class HttpMessageHandler : IDisposable
	{
		protected HttpMessageHandler()
		{
			if (Logging.On)
			Logging.Enter(Logging.Http, (object) this, ".ctor", (string) null);
			
			if (!Logging.On)
			return;
			
			Logging.Exit(Logging.Http, (object) this, ".ctor", (string) null);
		}
}
```

### Adding a Custom Response Header
```cs
public class XPoweredByHeaderHandler : DelegatingHandler
{
	const string XPOWEREDBYHEADER = "X-Powered-By";
	const string XPOWEREDBYVALUE = "ASP.NET Web API";
	
	protected override Task<HttpResponseMessage> SendAsync(
		HttpRequestMessage request, CancellationToken cancellationToken)
	{
		return base.SendAsync(request, cancellationToken).ContinueWith(
			(task) =>
			{
				HttpResponseMessage response = task.Result;
				response.Headers.Add(XPOWEREDBYHEADER, XPOWEREDBYVALUE);
				return response;
			}
		);
	}
}

//Registration of Two Custom Message Handlers in Global.asax.cs

public class WebApiApplication : System.Web.HttpApplication
{
	protected void Application_Start()
	{
		RouteConfig.RegisterRoutes(RouteTable.Routes);
		var config = GlobalConfiguration.Configuration;
		config.MessageHandlers.Add(new XHttpMethodOverrideHandler());
		config.MessageHandlers.Add(new XPoweredByHeaderHandler());
	}
}
```

### Per-Route Message Handlers - API Key Message Handler Implementation
```cs
public class ApiKeyProtectionMessageHandler : DelegatingHandler {
	protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request,
	CancellationToken cancellationToken) {
		IEnumerable<string> values;
		request.Headers.TryGetValues("apikey", out values);
		
		if (null != values && values.Count() == 1) {
			return base.SendAsync(request, cancellationToken);
		}
		
		var tcs = new TaskCompletionSource<HttpResponseMessage>();
		tcs.SetResult(new HttpResponseMessage(HttpStatusCode.Unauthorized) {
			ReasonPhrase = "API Key required."
		});
		return tcs.Task;
	}
}

routes.MapHttpRoute(
	name: "Secret Api",
	routeTemplate: "secretapi/{controller}/{id}",
	defaults: new {id = RouteParameter.Optional},
	constraints: null,
	handler: new ApiKeyProtectionMessageHandler() {
	InnerHandler =
	new HttpControllerDispatcher(GlobalConfiguration.Configuration)
});
```

## Media Type Formatters
//Product.cs
//Creating a CSV Media Formatter
```cs
public class Product
{
	public int Id { get; set; }
	public string Name { get; set; }
	public string Category { get; set; }
	public decimal Price { get; set; }
}
```


//ProductCsvFormatter.cs
```cs
public class ProductCsvFormatter : BufferedMediaTypeFormatter
{
	public ProductCsvFormatter()
	{
		// Add the supported media type.
		SupportedMediaTypes.Add(new MediaTypeHeaderValue("text/csv"));
		
		//Another example
		SupportedMediaTypes.Add(new MediaTypeHeaderValue("application/x.product"));
		
		//Supporting a Specific Encoding
		SupportedEncodings.Add(new UTF8Encoding(encoderShouldEmitUTF8Identifier: false));
		SupportedEncodings.Add(Encoding.GetEncoding("iso-8859-1"));
		
		//Using a MediaTypeMapping
		MediaTypeMappings.Add(new ProductMediaMapping());
	}
	
	public override bool CanWriteType(System.Type type)
	{
		if (type == typeof(Product))
		{
			return true;
		}
		else
		{
			Type enumerableType = typeof(IEnumerable<Product>);
			return enumerableType.IsAssignableFrom(type);
		}
	}

	public override bool CanReadType(Type type)
	{
		return false;
	}

	public override void WriteToStream(Type type, object value, Stream writeStream, HttpContent content)
	{
		using (var writer = new StreamWriter(writeStream))
		{
			var products = value as IEnumerable<Product>;
			if (products != null)
			{
				foreach (var product in products)
				{
					WriteItem(product, writer);
				}
			}
			else
			{
				var singleProduct = value as Product;
				if (singleProduct == null)
				{
					throw new InvalidOperationException("Cannot serialize type");
				}
				WriteItem(singleProduct, writer);
			}
		}
	}

	//Setting the HTTP Response Headers
	public override void SetDefaultContentHeaders(Type type, 	HttpContentHeaders headers, MediaTypeHeaderValue mediaType) {
		base.SetDefaultContentHeaders(type, headers, mediaType);
			headers.Add("X-ModelType", 	type == typeof(IEnumerable<Product>) ? "IEnumerable<Product>" : "Product");
			headers.Add("X-MediaType", mediaType.MediaType);
	}

	// Helper methods for serializing Products to CSV format. 
	private void WriteItem(Product product, StreamWriter writer)
	{
		writer.WriteLine("{0},{1},{2},{3}", Escape(product.Id),
			Escape(product.Name), Escape(product.Category), Escape(product.Price));
	}

	static char[] _specialChars = new char[] { ',', '\n', '\r', '"' };

	private string Escape(object o)
	{
		if (o == null)
		{
			return "";
		}
		string field = o.ToString();
		if (field.IndexOfAny(_specialChars) != -1)
		{
			// Delimit the entire field with quotes and replace embedded quotes with "".
			return String.Format("\"{0}\"", field.Replace("\"", "\"\""));
		}
		else return field;
	}
}


```

```cs
//Creating a Media Type Mapping
//ProductMediaMapping.cs
public class ProductMediaMapping : MediaTypeMapping {
	public ProductMediaMapping() : base("application/x.product") {
	}
	
	public override double TryMatchMediaType(HttpRequestMessage request) {
		IEnumerable<string> values;
		return request.Headers.TryGetValues("X-UseProductFormat", out values) && values.Where(x => x == "true").Count() > 0 ? 1 : 0;
	}
}
```

### Changing the Order of the Media Type Formatters
```cs
public static class WebApiConfig {
	public static void Register(HttpConfiguration config) {
		MediaTypeFormatter xmlFormatter = config.Formatters.XmlFormatter;
		config.Formatters.Remove(xmlFormatter);
		config.Formatters.Insert(0, xmlFormatter);
	}
}
```



### Disabling the Match-on-Type Feature
```cs
public static class WebApiConfig {
	public static void Register(HttpConfiguration config) {
		config.Services.Replace(typeof(IContentNegotiator),	new DefaultContentNegotiator(true));
	}
}
```

### Indenting the JSON Data
```cs
public static class WebApiConfig {
	public static void Register(HttpConfiguration config) {
		JsonMediaTypeFormatter jsonFormatter = config.Formatters.JsonFormatter;
		jsonFormatter.Indent = true;
		
		//Enabling the Microsoft Date Format
		jsonFormatter.SerializerSettings.DateFormatHandling = DateFormatHandling.MicrosoftDateFormat;
		
		//Enabling HTML Character Escaping
		jsonFormatter.SerializerSettings.StringEscapeHandling = StringEscapeHandling.EscapeHtml;
		
		//Ignoring Default Values
		jsonFormatter.SerializerSettings.DefaultValueHandling = DefaultValueHandling.Ignore;
	}
}
```


## Model Binding
```cs
//Registering the Media Type Formatter & Mapping Methods
//WebApiConfig.cs
//Registering a Media Type Formatter
public static class WebApiConfig {

	//Registering the Media Type Formatter
	public static void Register(HttpConfiguration config) {
		config.Formatters.Add(new ProductFormatter());
	}
	
	//Using Media Type Formatter Mapping Methods in the WebApiConfig.cs File
	MediaTypeFormatter prodFormatter = new ProductFormatter();
	prodFormatter.AddQueryStringMapping("format", "product", "application/x.product");
	prodFormatter.AddRequestHeaderMapping("X-UseProductFormat", "true", StringComparison.InvariantCultureIgnoreCase, false, "application/x.product");
	prodFormatter.AddUriPathExtensionMapping("custom", "application/x.product");
	config.Formatters.Add(prodFormatter);

}
```

## Input Validation

```cs
using System.ComponentModel.DataAnnotations;
namespace PartyInvites.Models {
	public class GuestResponse {
		[HttpBindNever]
		public int Id { get; set; }
		[Required]
		public string Name { get; set; }
		[Required]
		public string Email { get; set; }
		[Required]
		[Range(1, 100000)]
		public decimal Price { get; set; }
		[Required]
		public bool? WillAttend { get; set; }
	}
}


[ValidateModel]
public IHttpActionResult AddTask(HttpRequestMessage requestMessage, GuestResponse gr)
{
	var task = _processor.GuestResponse(gr);
	return result;
}

```

## Dependency Resolution
## Testing
## Optimization and Performance
## Hosting


## Filter
```cs
//Registering a Global Filter in the WebApiConfig.cs File
public static class WebApiConfig {
	public static void Register(HttpConfiguration config) {
		config.Filters.Add(new SayHelloAttribute { Message = "Global Filter" });
		
		config.MessageHandlers.Add(new AuthenticationDispatcher());
	}
}

//SayHelloAttribute
using System.Web.Http.Filters;
namespace Dispatch.Infrastructure {
	public class SayHelloAttribute : ActionFilterAttribute {
		public string Message { get; set; }
		
		public override Task OnActionExecutingAsync(HttpActionContext actionContext, CancellationToken cancellationToken) {
			Debug.WriteLine("SayHello: {0}", (object)Message ?? "Hello");
			return Task.FromResult<object>(null);
		}
	}
}


//Appling the Authorization Filter
namespace Dispatch.Controllers {
	[Time]
	public class ProductsController : ApiController {

		[CustomAuthentication]
		[CustomAuthorization("admins")]
		public Product Get(int id) {
			return products.Where(x => x.ProductID == id).FirstOrDefault();
		}
	}
}	
	
	
//Creating an Exception Filter
public class CustomExceptionAttribute : Attribute, IExceptionFilter {
	public Task ExecuteExceptionFilterAsync(HttpActionExecutedContext actionExecutedContext, CancellationToken cancellationToken) {
		if (actionExecutedContext.Exception != null && actionExecutedContext.Exception is ArgumentOutOfRangeException) {
			actionExecutedContext.Response = actionExecutedContext.Request.CreateErrorResponse( HttpStatusCode.BadRequest, "No data item");
		}
	
		return Task.FromResult<object>(null);
	}

	public bool AllowMultiple {
		get { return true; }
	}
}
```
## Error Handling
```cs
//Applying the HttpResponseException
[LogErrors]
public Product Get(int id) {
	Product product = products.Where(x => x.ProductID == id).FirstOrDefault();
		if (product == null) {
			throw new HttpResponseException(HttpStatusCode.BadRequest);
		}
	return product;
}


//Using an Implementation of the IHttpActionResult Interface
[LogErrors]
public IHttpActionResult Get(int id) {
	Product product = products.Where(x => x.ProductID == id).FirstOrDefault();
		if (product == null) {
			return BadRequest("No such data object");
		}
	return Ok(product);
}


//Creating an Error Response
[LogErrors]
public HttpResponseMessage Get(int id) {
	Product product = products.Where(x => x.ProductID == id).FirstOrDefault();
	if (product == null) {
		return Request.CreateErrorResponse(HttpStatusCode.BadRequest, new HttpError {
			Message = "No such data item",
			MessageDetail = string.Format("No item ID {0} was found", id)
		});
	}
	return Request.CreateResponse(product);
}


//Adding Extra Error Information
[LogErrors]
public HttpResponseMessage Get(int id) {
	Product product = products.Where(x => x.ProductID == id).FirstOrDefault();
	if (product == null) {
		HttpError error = new HttpError();
		error.Message = "No such data item";
		error.Add("RequestID", id);
		error.Add("AvailbleIDs", products.Select(x => x.ProductID));
		return Request.CreateErrorResponse(HttpStatusCode.BadRequest, error);
	}
	return Request.CreateResponse(product);
}

//Adding Model State Data to the Error
public HttpResponseMessage Post(Product product) {
	if (!ModelState.IsValid) {
		HttpError error = new HttpError(ModelState, false);
		error.Message = "Cannot Add Product";
		error.Add("AvailbleIDs", products.Select(x => x.ProductID));
		return Request.CreateErrorResponse(HttpStatusCode.BadRequest, error);
	}
	product.ProductID = products.Count + 1;
	products.Add(product);

	return Request.CreateResponse(product);
}


public static class WebApiConfig {
public static void Register(HttpConfiguration config) {
	//Setting the Exception Detail Policy
	config.IncludeErrorDetailPolicy = IncludeErrorDetailPolicy.Never;
	
	//Registering a Global Exception Handler
	config.Services.Replace(typeof(IExceptionHandler), new CustomExceptionHandler());
	
	//Registering a Custom Global Exception Logger
	config.Services.Add(typeof(IExceptionLogger), new CustomExceptionLogger());
}


//Throwing Exceptions
[LogErrors]
public Product Get(int id) {
	Product product = products.Where(x => x.ProductID == id).FirstOrDefault();
	if (product == null) {
		throw new ArgumentOutOfRangeException("id");
	}
		return product;
}
```
## Tracing and Logging
## API Documentation
