# WebAPI


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
