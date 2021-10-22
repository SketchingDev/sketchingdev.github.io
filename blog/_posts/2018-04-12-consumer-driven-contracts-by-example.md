---
layout: blog-post
title:  "Consumer-Driven Contracts by Example"
date:   2018-04-12 00:00:00
categories: spring testing contracts cdc
image-base: /assets/images/posts/2018-04-12-consumer-driven-contracts-by-example
---

As a developer for a distributed system, I work in a team that relies on the micro-services of other teams. We all beaver away in our separate teams safe in the knowledge that our great communication and nightly system tests will prevent any unforeseen breaking changes…

That was until one Tuesday morning when the peaceful tapping away of developers was shattered by the exclamation of two members of opposite teams who had just realised (that in spite of our frequent cross-team discussions, system tests and shiny Swagger specs!) a change had been merged to one of the APIs that broke our expectations of how it was to be consumed. Without anyone being aware of it, our teams had become apples and oranges!

So what might we have done to prevent such a dystopian scenario?

## Consumer-Driven Contracts

The problem was that the teams had no way to test the interactions of other services to theirs without either:

- Performing end-to-end tests of either a full or partial deployment of the system - a time intensive task.
- Mocking the other service - mocks will need updating to reflect changes to the service its representing

One solution to these problems is the Consumer-Driven Contracts pattern, which works on the basis of APIs exemplifying their interactions with one another in ‘contracts’. These contracts can then be exercised against the APIs to quickly highlight breaking changes, without the need to deploy other micro-services.

Let’s explore Consumer-Driven Contracts by implementing [Spring Cloud Contract](https://github.com/spring-cloud/spring-cloud-contract) into a simple project that consists of two APIs:

- [Image API](https://github.com/SketchingDev/Draw-by-Days/tree/361a00565eb52f12a57931a3ffd2add3eac91d50/image-api) - This is the **producer** of URIs to lovely images
- [Website](https://github.com/SketchingDev/Draw-by-Days/tree/361a00565eb52f12a57931a3ffd2add3eac91d50/website) - This is the **consumer** of the Image API and will display the images for aspiring artists to draw

![Website consumer and Image API producer]({{ page.image-base }}/consumer-and-producer.png)

Whilst developing features for the Website, I want to inform the Image API of the expectations I have towards it, which I can do in **contracts**. These files simply exemplify the website’s interactions with the API. They are made accessible to the API and communicated to the team working on it.

In my project these contracts live in the Image API’s test resources ([image-api/src/test/resources/contracts](https://github.com/SketchingDev/Draw-by-Days/tree/361a00565eb52f12a57931a3ffd2add3eac91d50/image-api/src/test/resources/contracts)), although they could just as easily live in a common location.

```groovy
org.springframework.cloud.contract.spec.Contract.make {
    request {
        method 'GET'
        url '/image'
    }
    response {
        status 200
        body("""
{
  "image": {
    "id": 1,
    "uri": "./images/1.png"
  },
  "nextImage": {
    "id": 2,
    "uri": "./images/2.png"
  }
}
  """)
    }
}
```

Now with contracts in place, Spring Cloud Contracts will do the following:

1. During the Image APIs integration test phase, it tests that the service responds to the requests according to the Website’s expectations (as defined in the contracts)
2. If these tests pass, then a [Wiremock](http://wiremock.org/) stub of the Image API is produced that responds to requests, again just like the contracts define
3. The Website is able to use the latest Image API stub in its tests to impersonate the API

![Consumer-driven contracts in use]({{ page.image-base }}/consumer-driven-contracts.png)

All of this can happen in fast-running tests and without either of the services, depending on the other being stood up. What’s great is the website now has confidence that the API fulfils its expectations, and every time these contracts are tested against the API, an updated stub is automatically produced that can be used by dependent services to stub out the API in tests.

The code below shows the Website using the Image API stub ([website/src/test/java/com/drawbydays/website/gallery/GalleryControllerTest.java](https://github.com/SketchingDev/Draw-by-Days/blob/361a00565eb52f12a57931a3ffd2add3eac91d50/website/src/test/java/com/drawbydays/website/gallery/GalleryControllerTest.java)).

```java
@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureMockMvc
@AutoConfigureStubRunner
@ActiveProfiles("test")
public class GalleryControllerTest {

  @Autowired
  private MockMvc mockMvc;

  @Autowired
  private PortSelectionInterceptor portSelectionInterceptor;

  @Autowired
  private StubFinder stubFinder;

  @Before
  public void setup() {
    final int port = stubFinder.findStubUrl("com.drawbydays", "image-api").getPort();
    portSelectionInterceptor.setPort(port);
  }

  @Test
  public void image_and_next_image_with_uris_are_passed_to_view() throws Exception {
    mockMvc.perform(get("/"))
            .andExpect(status().isOk())
            .andExpect(model().attribute("image", hasProperty("uri")))
            .andExpect(model().attribute("nextImage", hasProperty("uri")));
  }

  @Test
  public void image_and_next_image_with_uris_are_passed_to_view_when_id_valid() throws Exception {
    mockMvc.perform(get("/").param("id", "2"))
            .andExpect(status().isOk())
            .andExpect(model().attribute("image", hasProperty("uri")))
            .andExpect(model().attribute("nextImage", hasProperty("uri")));
  }

  @Test
  public void null_image_and_next_image_passed_to_view_when_id_invalid() throws Exception {
    mockMvc.perform(get("/").param("id", "invalid_id"))
            .andExpect(status().isOk())
            .andExpect(model().attribute("image", nullValue()))
            .andExpect(model().attribute("nextImage", nullValue()));
  }

  @TestConfiguration
  public static class GalleryControllerTestConfig {
    //...
  }
}
```

Hopefully this brief article has shown the usefulness of consumer-driven contracts to enrich cross-team communication and speed up feedback on breaking changes. For further reading, I would suggest Spring Cloud Contract’s excellent [step-by-step process for using consumer-driven contracts.](https://cloud.spring.io/spring-cloud-contract/single/spring-cloud-contract.html#_step_by_step_guide_to_consumer_driven_contracts_cdc)
