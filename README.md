# Instantiate Generic Parameter Classes With Dependencies
*Automatically instantiate dual-parameter (&lt;,>) classes with any dependencies in .NET Core app.*

## Intro
When you are using classes with two generic parameters (Class<T1, T2>), a big question is how to instantiate them and then be able to use them with dependency injection. Moreover, if a particular generic generic class has a dependency of its own, how do we dynamically instantiate them in turn. And to cap it all, at the consumer-side code, how do we abstract the implementation so it doesn't need to worry about any of the above.

This sort of situation can arise in mapper problems when you have to map one entity to another. You can use AutoMapper, but if you are using a customized solution or in my case, a hybrid solution involving both AutoMapper and custom mappers, you may need a seamless solution involving no complication at the caller code and clear distinction in the middle layer to know what to call and then lket mapping occur at the assigned place.

This work is done in .NET 5.0 and it's compatible with .NET Core versions 3.1 and 2.1. 

## Main Actors
As I said above, I am using a hybrid solution for mapping that involves both AutoMapper and a custom mapping solution. AutoMapper is not only used within my customized solution on certain occasions, but is also used via its Profile implementation. My custom solution implements an interface called ITransformer<T1, T2>

We have a consumer (like a web app) which is going to call a mapper service and provide it the types (source and destination) as well as object(s) (once again, source and destination) if available/applicable. Mapper service is going to figure out whether for given type, it needs a mapper implementing ITransformer<,> or one implementing Profile. It will call relevant instant and let it do the mapping.

To call 'relevant instance' as written above, it needs to have an instantiated class available to it. To do this, we go back to Startup when application is booting. Here, we will instatiate all ITransormer<,> based classes and also take care of any dependencies any particular class (implementing ITransformer<,>) may need.

## Step by Step

### The Interfaces

#### ITransformer<,>
The ITranformer<,> is a very simple interface exposing only one method: ```Transform```. Because at the end of the day, any class implementing this interface has only one thing to do, and that is to transform (or map) one entity to another. 

```
public interface ITransformer<T1, T2>
{
    T2 Transform(T1 input, T2 output);
}
```
Remember, the end consumer is not aware of this interface. It's only consumer is the Mapper Service that implements the IMapperService interface. 

#### IMapperService
The IMapperService is a consumer facing interface that exposes any methods that will be used by consumer end code to map objects. It also contains one method and its overloaded definition to allow users to map objects from source entity to destination.

```
public interface IMapperService
{
    T2 MapObjects<T1, T2>(T1 input) where T1 : class where T2 : class;

    T2 MapObjects<T1, T2>(T1 input, T2 output) where T1 : class where T2 : class;
}
```

### Startup
Now it's time for some action. First step, we need all ITransformer classes instantiated. In ```ConfigureServices``` method of ```Startup.cs```:

```
services.AddMemoryCache();
ReadAppSettings();
services.AddHttpContextAccessor();

var mapperTypes = Assembly.GetExecutingAssembly().GetExportedTypes();
var query = mapperTypes.Where(x => !x.GetTypeInfo().IsAbstract && !x.GetTypeInfo().IsGenericTypeDefinition)
    .Select(x => new { 
        matchingInterface = x.GetTypeInfo().ImplementedInterfaces
            .Where(i => i.GetTypeInfo().IsGenericType && i.GetGenericTypeDefinition() == typeof(ITransformer<,>)).FirstOrDefault(),
        tType = x
    })
    .Where(x => x.matchingInterface != null)
    .Select(x => new Tuple<Type, Type>(x.matchingInterface, x.tType));
    
// Alternate Linq Expression for easier understanding
var queryToo = from tType in mapperTypes
      where !tType.GetTypeInfo().IsAbstract && !tType.GetTypeInfo().IsGenericTypeDefinition
      let interfaces = tType.GetTypeInfo().ImplementedInterfaces
      let genericInterfaces = interfaces.Where(i => i.GetTypeInfo().IsGenericType && i.GetGenericTypeDefinition() == typeof(ITransformer<,>))
      let matchingInterface = genericInterfaces.FirstOrDefault()
      where matchingInterface != null
      select Tuple.Create(matchingInterface, tType);

foreach (var t in query)
{ 
    var lifetime = t.Item2.GetConstructors()[0].GetParameters().Count() == 0 ? ServiceLifetime.Singleton : ServiceLifetime.Scoped;
    services.Add(new ServiceDescriptor(t.Item1, t.Item2, lifetime));
    services.Add(new ServiceDescriptor(t.Item2, t.Item2, lifetime));
}                        
services.AddSingleton<IMapperService, MapperService>();
```

An ITransformer<,> class may need its own dependency to work. For example, if your ITransformer class is calling another service as DI in its constructor. Care must be taken in how you instantiate this class. If it's using a scoped dependency, and you instantiate it as Singleton, error will be encountered. It's not a bad idea to go with the lowest scope (Transient) if any of your depended-upon service is Transient. 

The code above will ensure that when Mapper Service looks for a specific ITransformer class, it finds it already instantiated in the correct scope and with the correct dependency(ies) loaded.

### Mapper Service
Mapper Service is the center stage of bringing in consumer code requests and then dispensing them to relevant mapper. Since we are doing a hybrid approach of using both cutomized ITransformer and AutoMapper's Profile maps, the service needs to keep track of all relevant maps which it does using two singleton properties (we don't need to find them everytime we run). Since Mapper Service itself is singleton, this all plays well together.    

```
public class MapperService : IMapperService
{
    protected readonly string _mappersAssembly = "MappersAssembly";
    protected IServiceScopeFactory _serviceScopeFactory;
    protected List<Type> _profileBasedMapperClasses;
    protected IMapper _profileBasedMaps;
    protected Dictionary<string, Type> _iTransformerMapTypes;        

    public IMapper ProfileBasedMaps
    {
        get => _profileBasedMaps ?? SetupProfileBasedMaps();
        set => _profileBasedMaps = value;
    }

    public Dictionary<string, Type> ITransformerMapTypes 
    { 
        get => _iTransformerMapTypes ?? GetITransformerTypes(); 
        set => _iTransformerMapTypes = value; 
    }

    public MapperService(IServiceScopeFactory ssf)
    {
        _serviceScopeFactory = ssf;
    }

    public T2 MapObjects<T1, T2>(T1 input) where T1 : class where T2 : class
    {
        return MapObjects<T1, T2>(input, null);
    }

    public T2 MapObjects<T1, T2>(T1 input, T2 output) where T1 : class where T2 : class
    {
        var key = $"{typeof(T1).FullName}{typeof(T2).FullName}";
        if (ITransformerMapTypes.ContainsKey(key))
        {
            using (var scope = _serviceScopeFactory.CreateScope())
            {
                var t = scope.ServiceProvider.GetService(ITransformerMapTypes[key]) as ITransformer<T1, T2>;
                return t.Transform(input, output);
            }
        }
        else
            try
            {
                return ProfileBasedMaps.Map<T1, T2>(input, output);
            }
            catch (AutoMapperMappingException) {  } 

        throw new ArgumentException(
            $"Mapper to resolve types {typeof(T1).Name} and {typeof(T2).Name} is either not found or has encoutered error. Hint: If your mapper" +
            $" is implementing Profile, throw the relevant exception in the associated block.");
    }

    private IMapper SetupProfileBasedMaps()
    {
        var profiles = Assembly.GetExecutingAssembly().GetTypes()
                .Where(x => x.IsSubclassOf(typeof(Profile))).ToList<Type>();

        var mappingConfig = new MapperConfiguration(mc =>
        {
            foreach (Type p in profiles)
                mc.AddProfile(p.GetConstructors()[0].Invoke(new object[] { }) as Profile);
        });
        _profileBasedMaps = mappingConfig.CreateMapper();

        return _profileBasedMaps;
    }

    private Dictionary<string, Type> GetITransformerTypes()
    {
        var tTypes = Assembly.GetExecutingAssembly().GetTypes()
                .Where(x => x.GetInterfaces().Any(y => y.Name.Contains("ITransformer") && y.GenericTypeArguments.Length > 0))
                .ToList<Type>();

        _iTransformerMapTypes = new Dictionary<string, Type>();
        foreach (Type t in tTypes)
        {
            var implInterface = t.GetInterfaces().FirstOrDefault(x => x.GenericTypeArguments.Length > 1);
            var type1Name = implInterface.GenericTypeArguments[0].UnderlyingSystemType.FullName;
            var type2Name = implInterface.GenericTypeArguments[1].UnderlyingSystemType.FullName;
            var key = $"{type1Name}{type2Name}";
            _iTransformerMapTypes.Add(key, t);
        }

        return _iTransformerMapTypes;
    }
}
```
The ```MapObjects``` method which is exposed to consumer tries to find an ITransformer method using a key created by the full names of the two types provided by the consumer within a dictionary of all ITransfomrer implementing classes. If there is an ITransformer that implements those types, it invokes its ```Transform``` method by getting the that class's instance from the scope it has just created. This makes sure no class and its dependency (if applicable) is being called in different scopes. If it doesn't find an ITransformer based class, it then refers to Profile based maps to find the relevant mapper. You must have a mapper in either of these two categories. If it doesn't find it, it only means that the mapper required to map the provided types is not implemented yet and it is described as such in the exception message. It can also allow the developer to debug any AutoMapper/Profile related error if the exception is specifically related to AutoMapper.

### Consumer

The consumer gets the abstraction in form of ```MapObjects```method exposed by ```IMapperService``` interface. All it needs to know is its source and destination entity types and their relevant objects. It is understood that it needs to have an acceptable source object. Whether it has the destination object created or not depends on the specific situation. Either or, it can use one of the overloaded ```MapObjects``` methods.

Let's say I have a ASP.NET web app and I need to map a domain entity (a POCO object of a database table) to a ViewModel entity in an MVC controller to display a list of courses undertaken by a student. For brevity, I have ignored other implementation details like DB ops.

```
public class CourseController : Controller
{
    protected IMapperService _mapperService;

    public DriverCourseController(IMapperService ms)
    {
        _mapperService = ms;
    }

    [HttpGet]
    public IActionResult Index(int studentId)
    {
        // Courses fetched from DB using Student ID
        IList<StudentCourse> studentCoursesFromDb;
        // Map the courses to a ViewModel class tailor-made for the view requirements
        var studentCoursesViewModel = _mapperService.MapObjects<IList<StudentCourse>, StudentCourseListViewModel>(studentCoursesFromDb);

        return View(vm);
    }
} 
```
Or if I have a course that is edited by user and I need to save it:

```
[HttpPost]
public IActionResult Update(StudentCourseViewModel viewModel)
{
    // DB entry of the student course undergoing update
    StudentCourse dbRecord;
    // Map the updates to the DB entry. Assign the newly mapped entity record to the same variable
    dbecord = _mapperService.MapObjects(vm, record);
    
    // Save the new record as an UPDATE.
    
    // Redirect back to GET implemntation of Update
    return RedirectToAction("Update", new {StudentId = viewModel.StudentId, CourseId = CourseId });
}
```

### A bit about ITransformer<,> class needing its own dependency
As I touched on before, a class implementing ITransforme<,> may need its ow dependency. Let's say while mapping objects, I need to check something from database and I don't have that information available to me. In that case, I would either need to use an abstraction just like MapperService or directly use the options exposed to me by my ORM. Since this article is not about that discussion, I'm simply going to show this idea as an abstraction called ```RepositoryService```. What I need to do is simply use DI to call ```RepositoryService``` here and use it as per need. Work done in ```Startup``` will do the rest. And it has to be intelligent enough to only provide DI for those classes that need it and also, provide those specific dependencies that are required. 

```
public class SourceEntityToDestinationEntityMapper : ITransformer<SourceEntity, DestinationEntity>
{
    protected IRepositoryService _repositoryService;

    public SourceEntityToDestinationEntityMapper(IRepositoryService rs)
    {
        _repositoryService = rs;
    }

    public DestinationEntity Transform(SourceEntity input, DestinationEntity output)
    {
        // Use RepositoryService to do required ops and take relevant mapping steps.

        return output;
    }
}
```

## Thus...
This mechanism gives me full flexibility of working with my mapping related tasks. Now I don't need to worry about anything but just the immediate mapping problem at hand and see if I should use Profile or ITransformer<,>. 
