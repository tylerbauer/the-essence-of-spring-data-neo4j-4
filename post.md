This guide will get you up and running with Spring Data Neo4j 4 in under an hour.

It is based on a live application, [Flavorwocky](http://www.flavorwocky.com) the winner of the [Neo4j Heroku Challenge 2012](http://blog.neo4j.org/2012/03/neo4j-heroku-challenge-winner-and.html). Rewritten to use Spring Data Neo4j 4, the code is open source and available on [Github](https://github.com/luanne/flavorwocky/tree/sdn).

## Introducing Spring Data Neo4j 4

[Neo4j](http://neo4j.com) is the world's most popular graph database. 
With ACID guarantees and the ability to scale to billions of nodes and relationships, Neo4j is the preferred choice for modelling highly connected domains.

[Spring Data Neo4j](http://projects.spring.io/spring-data-neo4j/) is part of the Spring Data initiative and simplifies development using annotations on simple POJO domain objects, much like JPA.

Spring Data Neo4j 4 supports Neo4j deployments in standalone server mode and has been rewritten from scratch by [GraphAware](http://graphaware.com), sponsored by [Neo Technology](http://neo4j.com/).

## Getting Started - The Graph Model
Before we write any code, we're going to model our domain as a graph. Flavorwocky is a very simple domain, perfect for this guide.

We have two entities- an Ingredient and a Category. An Ingredient belongs to a Category. An Ingredient also pairs with other Ingredients, with some degree of affinity. 
Here's what it looks like:

<img src="https://www.dropbox.com/s/bw6u56evfz07s4o/air-graph-model.png?dl=0">)

An Ingredient has a single relationship `HAS_CATEGORY` to a Category node. It also has potentially many `PAIRS_WITH` relationships to other Ingredients.

Flavorwocky keeps track of the last few pairings added to the graph. For simplicity, we're going to track this with another kind of node labelled `LatestPairing`. These nodes have no relations to any others. Better change tracking can be achieved by using something like the [GraphAware ChangeFeed Module](http://graphaware.com/neo4j/2014/08/27/graphaware-neo4j-changefeed.html) but that is outside the scope of this article.

## Setting up
Spring Data Neo4j 4 Milestone 1 was released in March 2015. RC1 is due to be released in the last week of May and contains a considerable amount of improvements and fixes. The dependencies included here will use the latest snapshot build so that we can use all these new features.

Include the following dependency for Spring Data Neo4j 4 in your project.
```xml
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-neo4j</artifactId>
    <version>4.0.0.BUILD-SNAPSHOT</version>
</dependency>
```
Specify the snapshot repository as well

```xml
<repository>
    <id>spring-libs-snapshot</id>
    <name>Spring</name>
    <url>http://repo.spring.io/libs-snapshot</url>
</repository>
```
## Domain Model

### NodeEntities, Relationships and RelationshipEntities

Nodes are modelled as simple POJO's with a few Spring Data Neo4j annotations; [Category](https://github.com/luanne/flavorwocky/blob/sdn/src/main/java/com/flavorwocky/domain/Category.java) is the simplest.

```java
@NodeEntity
public class Category {

	@GraphId
	private Long id;

	private String name;

	public Category() {
	}

	public Category(String name) {
		this.name = name;
	}
    
    //Getters and setters

}
```

Note the following-

* @NodeEntity indicates that this entity is backed by a Node in the graph. It is not mandatory to specify this annotation. The simple classname, in this case `Category` is used as the label for this entity. The label can be overridden with `@NodeEntity(label="FoodGroup")`
* Spring Data Neo4j tracks nodes by their Neo4j node id. Hence, it is mandatory to specify a field of type Long. If you have a field `Long id` then the `@GraphId` annotation is not required and the field will be used to represent the Neo4j node id.
* A public no arg constructor is required


Next, the [Ingredient](https://github.com/luanne/flavorwocky/blob/sdn/src/main/java/com/flavorwocky/domain/Ingredient.java).

```java
@NodeEntity
public class Ingredient {

	private Long id;
	private String name;

	@Relationship(type = "HAS_CATEGORY", direction = "OUTGOING")
	private Category category;

	@Relationship(type = "PAIRS_WITH", direction = "UNDIRECTED")
	private Set<Pairing> pairings = new HashSet<>();

	public Ingredient() {
	}

	public Ingredient(String name) {
		this.name = name;
	}

	public void addPairing(Pairing pairing) {
		pairing.getFirst().getPairings().add(pairing);
		pairing.getSecond().getPairings().add(pairing);
	}
	
	//Getters and setters
}

```

Again, this entity is backed by a Node, indicated by the `@NodeEntity`. The `Long id` serves as the node id.
An ingredient has a relationship to a category, indicated here as
```java
@Relationship(type = "HAS_CATEGORY", direction = "OUTGOING")
private Category category;
```

This tells Spring Data Neo4j three things - 

* A relationship is to be maintained between the Ingredient and Category nodes
* The relationship type is called `HAS_CATEGORY`
* The direction of the relation is outgoing from the Ingredient to the Category

The `@Relationship` annotation is not mandatory, in which case the relationship type will be derived from the property name and upper snake cased i.e. `CATEGORY`. The direction if omitted defaults to Outgoing.

The second relationship, `PAIRS_WITH` is a special kind of relationship because it has properties on it.

```java
@Relationship(type = "PAIRS_WITH", direction = "UNDIRECTED")
private Set<Pairing> pairings = new HashSet<>();
```
Neo4j is a labelled property graph, so both nodes and relationships can have properties. We want to store the affinity between two ingredients as a property on the `PAIRS_WITH` relationship. We also do not care about the direction of the `PAIRS_WITH` relationship between two ingredient nodes, so the direction specified is `UNDIRECTED`, which means it can be traversed from either direction. 

This relationship is now modelled in our domain as a Relationship Entity, [Pairing](https://github.com/luanne/flavorwocky/blob/sdn/src/main/java/com/flavorwocky/domain/Pairing.java).


```java
@RelationshipEntity(type = "PAIRS_WITH")
public class Pairing {

	Long id;

	@StartNode
	private Ingredient first;
	@EndNode
	private Ingredient second;
	private Affinity affinity;


	public Pairing() {
	}

    //Getters and setters
}
```

The `@RelationshipEntity` annotation is mandatory along with the relationship type. Also note that the relationship type `PAIRS_WITH` is mandatory on the `pairings` field in the Ingredient class, since it represents a relationship entity.
Also mandatory are the `@StartNode` and `@EndNode` indicating the start and end nodes of the relationship. 
As usual, we require a `Long id` which serves as the relationship id.
Apart from this, we can define as many properties as we like.

Note that when we add a Pairing via the `addPairing` method, we make sure that we set it on both ingredients comprising the pair. This ensures we can navigate from both ends of the relationship and hence save either entity correctly.


Representing a [LatestPairing](https://github.com/luanne/flavorwocky/blob/sdn/src/main/java/com/flavorwocky/domain/LatestPairing.java) is easy enough.

```java
@NodeEntity
public class LatestPairing {

	Long id;

	@DateLong
	Date dateAdded;
	String ingredient1;
	String ingredient2;

	public LatestPairing() {
	}

    //Getters and setters
}
```

### Transient entities

If you have objects which are not to be persisted to the graph, annotate them with `@Transient`. This annotation applies to both classes and fields. 

## Converters
Neo4j supports the following types of property values- numeric, boolean, String and arrays of them.

Spring Data Neo4j 4 includes a set of default type converters to deal with types such as Dates, BigInteger, BigDecimal, byte[] and Enum.

This means we can have a Date field defined in our entity and have it converted to a Long when persisted, and back to a Date when retrieved.

```java
@DateLong
Date dateAdded;
```
By default, with no annotation, Dates will be stored as Strings. The date format can be customized with the `@DateString` annotation.

Enums are also converted to Strings and back automatically, so defining 

```java
private Affinity affinity;
```

will work just fine. 

Spring Data Neo4j 4 also supports custom converters with the `@Convert` annotation backed by an implementation of `org.neo4j.ogm.typeconversion.AttributeConverter`.

## Repositories

Repositories build on the composable repository infrastructure provided by Spring Data Commons.
In order to get for free the ability to save, delete or find entity instances, we'll define our [repository interfaces](https://github.com/luanne/flavorwocky/tree/sdn/src/main/java/com/flavorwocky/repository) to inherit from `GraphRepository<T>`

Here is what they look like.
```java
@Repository
public interface LatestPairingRepository extends GraphRepository<LatestPairing> {

}
```
We do not require any additional functionality for LatestPairings, the save and find we get by virtue of extending `GraphRepository` are sufficient.

However, the IngredientRepository is a little more involved.

```java
@Repository
public interface IngredientRepository extends GraphRepository<Ingredient> {

	List<Ingredient> findByName(String name);

	@Query("match p=(i:Ingredient {name:{0}})-[r:PAIRS_WITH*0..3]-(i2)-[:HAS_CATEGORY]->(cat) return p;")
	Iterable<Map<String, Object>> getFlavorPaths(String ingredientName);

	@Query("match (ing1:Ingredient {name: {0}})-[r1:PAIRS_WITH]-(ing2)-[r2:PAIRS_WITH]-(ing3)-[r3:PAIRS_WITH]-(ing1) return ing1.name as firstName, ing2.name as secondName,ing3.name as thirdName, ID(r2) as relId")
	Iterable<Map<String, Object>> getTrios(String ingredient);
}
```

Spring Data Neo4j 4 supports queries derived from finder methods.
For example, simply specifying 

```java
List<Ingredient> findByName(String name);
```

is enough to have a Cypher query executed behind the scenes that filters by the name property on the Ingredient node and return a list of those that match.

Similarly, you can define finders such as `findByNameAndCategoryName` which will filter ingredients based on the name property of the Category to which they are related. You want to make sure that the order of parameters matches the order of expressions in the method name.

We can also execute an arbitrary Cypher query via the repository.
Specify the Cypher query in the `@Query` annotation and you're good to go.
In the example above, the results of the query are mapped to an Iterable of rows, where each row is represented as a `Map<String,Object>`. 
If you want to map results to an arbitrary class, use a class annotated with `@QueryResult` as the method return type and Spring Data Neo4j will apply the same simple mapping strategy as it does for normal entities.

With this, we're ready to start saving and retrieving our entities.

## Stirring in business logic

[PairingService](https://github.com/luanne/flavorwocky/blob/sdn/src/main/java/com/flavorwocky/service/PairingService.java) is going to contain our business logic.

### Fetching all ingredient names

The new object mapping framework in Spring Data Neo4j 4 introduces the concept of a persistence horizon which gives you fine grained control over the persistence depth.
A depth of 0 will load only the objects properties but not relations. A depth of 1 will load the object and its immediate neighbours. 
The default persistence depth is 1.

Flavorwocky retrieves all ingredient names to populate the type ahead on the search box. 
It is as simple as

```java
@Autowired
IngredientRepository ingredientRepository;
	
public Iterable<Ingredient> getIngredientNames() {
	return ingredientRepository.findAll(0);
}
```
Auto-wire the required Graph Repositories into Spring beans and then use the convenient `findAll()`. Note that in this case, we want only the ingredient names and not all their pairings, so we're overriding the default depth with a zero.

### Latest Pairings, sorted

Flavorwocky displays the last few pairings added to the graph. Recall that we model these with the `LatestPairing` entity which contains a `dateAdded` property.
Using the sorting features of Spring Data Neo4j, we don't have to worry about writing a custom Cypher query or sorting them in application code. We'll use the `loadAll` functionality on the `Session` interface.

```java
@Autowired Session neo4jSession;

public Collection<LatestPairing> getLatestPairings() {
		return neo4jSession.loadAll(LatestPairing.class,
		       new SortOrder().add(SortOrder.Direction.DESC, "dateAdded"),                 new Pagination(0, 5), 0);
	}

```
Here, we specify that we want to sort by the `dateAdded` property in descending order. We only want up to 5 latest pairings, so we provide a `Pagination` with page number 0 and page size 5.
That was very simple!

### Pairing ingredients

Saving entities is a matter of calling `save()` on the repository or the Session. When objects are persisted, the default depth is -1, or infinite. In other words, saving an object saves every object that is reachable from it. However, this isn't as alarming as it sounds because the Object Graph Mapper is able to detect which objects have been modified and require to be saved, ignoring the rest.

So let's create a couple of ingredients and pair them.

```java
//Set up the categories
Category meat = new Category("Meat");
Category veg = new Category("Vegetable");

//Set up two ingredients
Ingredient chicken = new Ingredient("Chicken");
chicken.setCategory(meat);

Ingredient carrot = new Ingredient("Carrot");
carrot.setCategory(veg);

//Pair them
Pairing pairing = new Pairing();
pairing.setFirst(chicken);
pairing.setSecond(carrot);
pairing.setAffinity(Affinity.EXCELLENT);
carrot.addPairing(pairing);

//Save
ingredientRepository.save(chicken);
```
Saving either ingredient will save every object reachable from it that has been modified, which in this case involves both ingredients, their categories and the pairing.


## Configuration
Now for the Spring configuration. Spring Data Neo4j 4 currently supports only Javabean based configuration.
[Application.java](https://github.com/luanne/flavorwocky/blob/sdn/src/main/java/com/flavorwocky/Application.java) shows how it's done.

```java
@Configuration
@ComponentScan("com.flavorwocky")
@EnableAutoConfiguration
@EnableNeo4jRepositories("com.flavorwocky.repository")
public class Application extends Neo4jConfiguration {

	@Override
	public Neo4jServer neo4jServer() {
		return new RemoteServer("http://localhost:7474");
	}


	@Override
	public SessionFactory getSessionFactory() {
		return new SessionFactory("com.flavorwocky.domain");
	}

	@Override
	@Bean
	@Scope(value = "session", proxyMode = ScopedProxyMode.TARGET_CLASS)
	public Session getSession() throws Exception {
		return super.getSession();
	}
}
```
First, your class must extend `Neo4jConfiguration`.
Specify where your repositories are located in the `@EnableNeo4jRepositories` annotation. 

The database URL is provided to the `RemoteServer` in the overridden `neo4jServer()` mthod.
The other variation of the `Neo4jServer` is an `InProcessServer` useful for testing.

The `SessionFactory` creates instances of `org.neo4j.ogm.session.Session` as required and sets up the object-graph mapping metadata when constructed. Provide the packages containing domain objects to the constructor of the `SessionFactory`.

Finally, specify the scope of the `Session`, which for our web application, is Session scope. The scope is important because it keeps track of changes made to entities and their relationships.
Only those which have changed are persisted to the graph. However there is no risk of getting stale data on load because the Session never returns cached data and always hits the database.

## Testing
Writing tests is fairly straightforward.

Include the following dependencies

```xml
 <dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-neo4j</artifactId>
    <version>4.0.0.BUILD-SNAPSHOT</version>
    <type>test-jar</type>
</dependency>

<dependency>
    <groupId>org.neo4j</groupId>
    <artifactId>neo4j-kernel</artifactId>
    <version>${neo4j.version}</version>
    <type>test-jar</type>
</dependency>

<dependency>
    <groupId>org.neo4j.app</groupId>
    <artifactId>neo4j-server</artifactId>
    <version>${neo4j.version}</version>
    <type>test-jar</type>
</dependency>
 <dependency>
    <groupId>org.neo4j</groupId>
    <artifactId>neo4j-io</artifactId>
    <version>${neo4j.version}</version>
    <type>test-jar</type>
</dependency>
```
[PersistenceContext.java](https://github.com/luanne/flavorwocky/blob/sdn/src/test/java/com/flavorwocky/context/PersistenceContext.java) sets up the Spring configuration for tests. It is essentially the same as the one we saw earlier, except that it uses an `InProcessServer`. This starts a new instance of `CommunityNeoServer` running on an available local port and returns the URL needed to connect to it.

Now simply instruct your test class to use this configuration, autowire in repositories as required, and you're ready to write some tests.

```java
@ContextConfiguration(classes = {PersistenceContext.class})
@RunWith(SpringJUnit4ClassRunner.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class DomainTest {

	@Autowired
	IngredientRepository ingredientRepository;
	
	@Test
	public void shouldBeAbleToSaveAnIngredient() {
		Category category = new Category("Dairy");
		Ingredient ingredient = new Ingredient("Emmental");
		ingredient.setCategory(category);
		ingredientRepository.save(ingredient);

		Ingredient emmental = IteratorUtil.firstOrNull(ingredientRepository.findByName("Emmental"));
		assertNotNull(emmental);
		assertEquals(ingredient.getName(), emmental.getName());
		assertEquals(category, emmental.getCategory());
	}
}
```
That's how easy it is to build an application using Spring Data Neo4j!

## Further Reading

Milestone 1 of Spring Data Neo4j has been released, and the documentation is available at http://docs.spring.io/spring-data/neo4j/docs/4.0.0.M1

RC1 is scheduled to be released in the last week of May 2015, the documentation will be published to http://docs.spring.io/spring-data/neo4j/docs/4.0.0.RC1

The source code for Flavorwocky is available at https://github.com/luanne/flavorwocky/tree/sdn
Instructions on running this locally are documented in the [README](https://github.com/luanne/flavorwocky/blob/sdn/README.md).

If you need help, please post a question on [StackOverflow](http://stackoverflow.com) and tag it with [spring-data-neo4j-4](http://stackoverflow.com/questions/tagged/spring-data-neo4j-4)


