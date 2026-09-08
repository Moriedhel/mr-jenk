--- Annotation ---

Annotations automatically generate code for you. 
They are used to reduce boilerplate 
code and make your code cleaner and more readable.

some important annotations

@Data 
Give to a class gets getters 
and setters automatically
(plus toString, equals, hashCode).  Methods 

@Builder readable way to create objects 
step by step instead of a long constructor.

EXAMPLE

Without builder:
new Product(null, sellerId, name, description, price, stock, imageUrls, null, null, null)


With builder:
Product product = Product.builder()
.sellerId(sellerId)
.name(name)
.description(description)
.price(price)
.stock(stock)
.imageUrls(imageUrls)
.build();

---- MongoRepository----

WHY we use it and how it is used to our product service
we create an interface that extends MongoRepository 

and we specify the type of the entity and the type of the id

so for example if we have a Product entity with an id of type String

we can create a ProductRepository like this:

public interface ProductRepository extends MongoRepository<Product, String> {
its a mapping between the entity and the database




Mongo repository  has methods we dont see

save(S entity)
•
findById(ID id)
•
findAll()
•
existsById(ID id)
•
deleteById(ID id)
•
count()

so we don't need to right querries for simple operations
we can add custom queries by writing methods with 
specific names and spring will generate the query for us

for example if we have a ProductRepository 
that extends MongoRepository<Product, String> 
we can add a method like this:
stream<Product> findBySellerId(String sellerId);
and spring will generate the query 
to find all products by sellerId



--- SPRING BEANS---

A Spring bean is an object created and managed by Spring’s container (ApplicationContext).
Instead of you doing:
ProductService s = new ProductService(...);
Spring does it for you, keeps lifecycle/state, and injects dependencies automatically.
Examples of beans in your app:
•
ProductService (@Service)
•
ProductController (@RestController)
•
ProductRepository (auto-created from MongoRepository interface)
•
SecurityFilterChain (from @Bean method)
So: bean = “Spring-managed object.”








---Transactional---  
tells Spring: run this method inside a database transaction.
Why we use it:

If everything works → save changes.
•
If something fails in the middle → cancel changes.
readOnly = true means: this method only reads data (no updates).
Without readOnly means: this method can write/update/delete data.



--- Controllers--- 

ResponseEntity is a Spring class that represents an HTTP response,
including status code, headers, and body.
It allows you to customize the response sent back to the client.

Annotations 

•
@RequestBody = “take JSON from the request and put it into this Java object.”
•
@Valid = “check this object rules first in DTO (required fields, positive price, etc.). If invalid, stop and return 400.”

@AuthenticationPrincipal Jwt jwt

**Give me the logged-in user token in this variable (jwt).”
Then you can read:
•
jwt.getSubject() -> user id
•
jwt.getClaim("role") -> user role (like SELLER)


---Security---


--KAFKA-- 

Cluster : the whole kafka installation, all brokers together
Broker : a single kafka server, part of the cluster
topic : a named channel for messages (like a mailbox)
partition : a topic can be split into multiple partitions for parallelism
producer : a service that sends messages to a topic
consumer : a service that reads messages from a topic




Analogy
Kafka	Real world
Cluster	A post office building
Broker	A worker in the post office
Topic	A named mailbox slot (e.g. "Media Events")
Partition	The physical pile of letters in that slot
Message	One letter
Producer	The person dropping a letter in
Consumer	The person picking letters up






Producer (Media Service)          Topic               Consumer (any service)
─────────────────────────         ──────────────      ──────────────────────
"image was uploaded"    ───────►  image.uploaded  ──► Product Service reads it
                                                  ──► Audit Service reads it
                                                  ──► Thumbnail Service reads it

                                                  One producer writes to the topic.
                                                   Many consumers can read from it
                                                    independently — each one gets
                                                     every message, at its own pace.

Messages are distributed across partitions by their key 
(the mediaId we talked about)
Same key always → same partition → guaranteed ordering per image
More partitions = more consumers can read in parallel


YML configuration for Kafka topics:
key      → "which mailbox" → String (mediaId)
value    → "the letter"    → JSON (our event object)
acks all → "I won't stop until everyone has it"
idempotence → "if I retry, don't deliver twice"


Kafka config.java

partitions	3	3+ (more = more parallel consumers)
replicas	1	3 (one copy per Kafka broker, survives broker crash)

Service starts
    └─ Spring finds KafkaAdmin + NewTopic beans
           └─ calls Kafka Admin API
                  └─ "create image.uploaded (3 partitions)" → already exists? skip. new? create.



ImageEventProducer.java
Spring Kafka template sends messages to topic:

publishImageUploaded(MediaAsset asset)   // called after a successful upload
publishImageDeleted(MediaAsset asset)    // called after a successful delete


topic — which channel to write to ("image.uploaded" or "image.deleted")
event.mediaId() — the partition key — guarantees all events for the same image land in the same partition in order
event — the ImageEvent record, serialized to JSON automatically


--IMPORTANT--
.whenComplete((result, ex) -> {
    if (ex != null) {
        log.error("Kafka publish failed ...");   // log but don't crash
    } else {
        log.debug("Kafka publish ok ...");
    }
});

IF service fails to publish to Kafka,
 we log the error but don't crash the service.


 --KafkaListener.java--
Spring Kafka listener reads messages from topic:  

Topics are in application.yml (or .properties) 
so we can change them without changing code.

you can see every topic that relates to the service there

Kafka listener is a method annotated with @KafkaListener
 that listens to a specific topic 
  and processes incoming messages.


@KafkaListener(topics = "${app.kafka.topics.image-deleted}", groupId = "product-service")
    public void onImageDeleted(ImageDeletedEvent event) {
        log.debug("Received IMAGE_DELETED event: mediaId={} sellerId={}", event.mediaId(), event.sellerId());

        List<Product> affected = productRepository.findByImageUrlsContaining(event.mediaId());
        if (affected.isEmpty()) return;




In our case for example we need to listen to the image.uploaded topic 
and when we receive a message we need to update the product 
with the new image url or Delete the image url if we receive
 a message from the image.deleted topic.


--- DISCOVERY SERVICE AND API GATEWAY ---

The Discovery Service is an internal registry for the backend services.
User Service, Product Service, and Media Service register their service name
and network location with it when they start.

The API Gateway is the external entry point for frontend and API requests.
Instead of using hard-coded service addresses, the gateway asks the Discovery
Service where a named service is running and then forwards the request to it.

Request flow:

Browser -> API Gateway -> Discovery lookup -> User/Product/Media Service

The Discovery Service does not expose the APIs to users. It only helps the
gateway find healthy internal services. The gateway is responsible for exposing
routes and applying cross-cutting rules such as authentication and CORS.





