---
title: Cancellation Tokens
category: code-tips
tags: language structure c-sharp
---

`System.Threading` expose an elegant model for 'coordinating the canceling of
asynchronous operations'. It's basic use is as simple as shown in the following
snippet.

```csharp
/// <summary>
/// Fetch payment given <paymentid>, will throw TaskCanceledException if
/// the operation takes langer than the given time.
/// </summary>
public async Task<Payment> QueryPayment(Guid paymentId, int timeoutInMilliseconds)
{
    // Setup a source that cancels after <timeout>
    using var source = new CancellationTokenSource(Timespan.FromMilliseconds(timeoutInMilliseconds));

    return await _repo.query(paymentId, source.Token);
}
```

A lot of libraries, including `System.Threading.Tasks.Task` operations, take a
cancellation token and handles the cancellation for you. It's also possible to
handle it yourself using the methods `<IsCancellationRequested>` and
`<ThrowIfCancellationRequested>`, just make sure to clean up anything before you
return.

Further more; .Net controllers take an optional source token parameter  by
default , which will cancel if the caller prematurely close the connection. It
seems to me that this is an often overlooked feature, which can be used to great
benefit in an API to reduce resource consumption and improve performance!

It applies, of course, only for endpoints that can be cancelled, e.g. read-only
operations.

```csharp
[HttpGet("{paymentId:guid}")]
public async Task<IActionResult> GetPayment(Guid paymentId, CancellationToken token)
{
    // Warning: potentially very slow and expensive operation.
    var rating = await repo.QueryCreditRating(paymentId, cancellationToken);

    return Ok(new { Rating = rating.value });
}
```

### Additional notes

* TaskCanceledException, which is thrown from most Task operations when the
  token requests cancellation, derives from the more generally used
  OperationCanceledException.

* Multiple source warn the developer to be cautious of the \point of no
  cancellation\, i.e. you will not always be able to uncritically shut down an
  operation.

* It's advised to make sure your cancellation token source is disposed when
  you're done using it, since the garbage collector will often be slow in doing
  it for you, e.g. by using the `using` keyword.

  ```csharp
  public async Task<Payment> GetPayment(Guid paymentId)
  {
    using var source = new CancellationTokenSource(TimeSpan.FromMilliseconds(100));
    var result = await repo.Query(..., source);
    return new Payment(...);
  }
  ```

* If you would like to look into how to hook up a global handle of
  OperationCancelledExceptions, for example to map them automatically to a HTTP
  response if they bubble up to the controller layer, have a look at `Andrew
  Locks` blog in the sources below.

### Code Examples and Extracts

* [CancellationTokenTests](https://github.com/tugend/CodeSamples/blob/master/CancellationTokenSamples/Tests/CancellationTokenTests.cs):
  A simple example of using CancellationTokens to manage competing queries

  ```csharp
  [Fact]
  public async Task RacingQueries()
  {
      // ARRANGE
      var userId = Guid.NewGuid();
      var repo = new PaymentRepository();
      using var source = new CancellationTokenSource();

      // Start two competing queries
      var racingQuery1 = repo.QueryByUserId(userId, source.Token);
      var racingQuery2 = repo.QueryByUserId(userId, source.Token);

      // the first task to complete is assigned <winner>, the other is assigned <looser>
      var winner = await Task.WhenAny(racingQuery1, racingQuery2);
      var looser = new[] {racingQuery1, racingQuery2}.Single(x => x != winner);

      // ACT
      // when we have found a winner, we cancel any still running tasks if any
      source.Cancel();

      // ASSERT

      // Winner completed successfully
      Assert.True(winner.IsCompleted);
      Assert.True(winner.IsCompletedSuccessfully);

      // Looser completed by cancellation
      Assert.True(looser.IsCompleted);
      Assert.True(looser.IsCanceled);

      // Winner was not cancelled
      Assert.False(winner.IsCanceled);

      // None where faulted
      Assert.False(winner.IsFaulted);
      Assert.False(looser.IsFaulted);
  }
  ```

* [AdaptivePaymentRepositoryTests](https://github.com/tugend/CodeSamples/blob/master/CancellationTokenSamples/Tests/AdaptivePaymentRepositoryTests.cs):
  An implementation of a repository that use cancellation tokens to gracefully
  degrade to the best possible service given a dynamic limit of execution
  time, i.e. it returns a more detailed result if it has the computation time.

  ```csharp
    /// <summary>
    /// Assume we have plenty of processor power but suffer from periodic slow reads.
    /// To introduce a gradual degradation of service rather than risk downtime,
    /// we could try to have a two tier read operation that fetch both the slow and
    /// the fast data, and return the most detailed information we have within
    /// the given timeout.
    /// </summary>
    public async Task<Payment> QueryPayment(Guid paymentId, CancellationToken token)
    {
        // Start two tasks that both reference the expiring cancellation token
        var shallowDetailedPaymentTask = _shallowPaymentRepository.QueryByUserId(paymentId, token);
        var detailedPaymentTask = _detailedPaymentRepository.QueryByUserId(paymentId, token);

        try
        {
            var shallowPayment = await shallowDetailedPaymentTask;
            try
            {
                // Yes! This user is going to be real happy,
                // we can show him all the details we have on the payment
                // without exceeding the given time!
                return Payment.FromDetailedSource(shallowPayment, await detailedPaymentTask);
            }
            catch (OperationCanceledException)
            {
                // Haha, we didn't have time to fetch all the details but we got the basis content.
                return Payment.FromShallowSource(shallowPayment);
            }
        }
        catch (OperationCanceledException)
        {
            // Haha, we didn't have time to fetch all the details but we got the basis content.
            throw new OperationCanceledException("Sorry, we didn't manage to get any results in time!");
        }
    }
    ```

* [WebAPITests](https://github.com/tugend/CodeSamples/blob/master/CancellationTokenSamples/Tests/WebApiTests.cs):
  An API controller implementation that show how using
  cancellation tokens can improve performance in an API where requests share a
  limited resource pool.

### Sources

* [System.Threading.CancellationTokenSource documentation](https://docs.microsoft.com/en-us/dotnet/api/system.threading.cancellationtokensource?view=net-5.0)

* [Using CancellationTokens in ASP.NET Core MVC controllers 2017 ~ Andrew Lock](https://andrewlock.net/using-cancellationtokens-in-asp-net-core-mvc-controllers/)

* [Recommended patterns for CancellationToken 2014 ~ Andrew Arnott](https://devblogs.microsoft.com/premier-developer/recommended-patterns-for-cancellationtoken/)


<!-- ---
<br />

## API Versioning with Swagger

...

  ## Swagger

  ### Versioning

  Best practice allow any caller of your API to easily verify whether he or she
  is using the right version of the api. Unfortunately developers often
  incrementally add new versions to select endpoints resulting in something like
  below. The confusion increase further when you have to figure out whether some
  endpoint versions must follow each other.

  ```
POST api/payments // create PUT api/payments // update GET
api/payments/{paymentId:guid} // fetch

PUT api/v2/payments

POST api/v3/payments // the newer way of creating a payment, please use this one
instead PUT api/v3/payments // the newer way of updating a payment, must be used
if payment was created using v3 endpoint
```

A better approach is to version the ENTIRE API such that any endpoints that are
unchanged from e.g. version 1 to version 3 just exists for both versions. The
consumer of the API will then be easily able to determine which endpoints to
use.

As a side note, there should of course be a change log for each version too.

Best practice for this is easy using swagger as the follow example show.

#### Examples

See PaymentsController.

```csharp
    [ApiController]
    [Route("api/v{version:apiVersion}/[controller]")]
    public class PaymentsControllerLegacyV1 : ControllerBase
    {
        [ApiVersion("1.0", Deprecated = true)]
        [HttpGet("{paymentId:guid}")]
        public async Task<IActionResult> Create(Guid paymentId, Payment CreatePayment, CancellationToken cancellationToken)
        ...
    }

    [ApiController]
    [Route("api/v{version:apiVersion}/[controller]")]
    public class PaymentsController : ControllerBase
    {
        [ApiVersion("2.0")]
        [ApiVersion("1.0")] // Unchanged from v1 to v2
        [HttpGet("{paymentId:guid}")]
        public async Task<IActionResult> Get(Guid paymentId, CancellationToken cancellationToken)
        {
            ...
        }

        [ApiVersion("2.0")]
        [HttpGet("{paymentId:guid}")]
        public async Task<IActionResult> Create(Guid paymentId, Payment CreatePayment, CancellationToken cancellationToken)
```


### Endpoint documentation with good examples

Good documentation is something any consumer of an API can really appreciate.

Swagger can render your remarks as neat markdown, and add your comments to the
response codes. The value given in the parameter as an example is further
expressed as a default value when calling the endpoint!


```csharp
    /// <summary>
        /// Endpoint for creating a payment
        /// </summary>
        /// <param name="paymentId">fc9d9f16-f775-43ac-8cde-99206a461809</param>
        /// <param name="speed">fast</param>
        /// <param name="cancellationToken"></param>
        /// <returns></returns>
        /// <remarks>
        ///
        /// Possible 'speed' values could be:
        ///
        ///     "fast", "slow", none
        ///
        /// Just for demonstration
        ///
        ///     POST api/v1/payments/fc9d9f16-f775-43ac-8cde-99206a461809?speed=fast
        ///     {
        ///     }
        ///</remarks>
        /// <response code="200">Returns result of query</response>
        /// <response code="400">Unknown payment</response>
        [ProducesResponseType(StatusCodes.Status410Gone)]
        [HttpGet("{paymentId:guid}")]
        public async Task<IActionResult> Get(Guid paymentId, string speed, CancellationToken cancellationToken)
        {
            var rating = speed?.Equals("fast") ?? false
                ? await repo.FastQueryCreditRating(paymentId, cancellationToken)
                : await repo.SlowQueryCreditRating(paymentId, cancellationToken);

            return Ok(new
            {
                Rating = rating.value
            });
        }
```

Continuing in the same track, it would be really nice if there was an actually
working example of input data that could just be fired off to test the endpoint.
Just add an example doc to the request object and the values are automatically
added. If you also want a new valid guid or similar dynamic value, you'll need
to a little bit more though.

```csharp
public class DeletePayment : IRequest
    {
        /// <summary>The amount of the product</summary>
        /// <example>200</example>
        public decimal Amount { get; set; }

        /// <summary>The description of the product</summary>
        /// <example>Men's basketball shoes</example>
        public string Description { get; set; }

        /// <summary>Type of purchase</summary>
        /// <example>Everyday needs</example>
        public string Type { get; set; }
    }
```

## Sources

* [ASPNET Core WebApi Project
  Essentials](https://dev.to/moesmp/what-every-asp-net-core-web-api-project-needs-part-1-serilog-o5a)

* [CancellationTokenSource](https://docs.microsoft.com/en-us/dotnet/api/system.threading.cancellationtokensource?view=net-5.0)
* [Andrew Lock: Using Cancellation Tokens
  ..](https://andrewlock.net/using-cancellationtokens-in-asp-net-core-mvc-controllers/)
* [Devblogs: Best Practice Cancellation
  Tokens](https://devblogs.microsoft.com/premier-developer/recommended-patterns-for-cancellationtoken/)

* https://bryanavery.co.uk/asynchronous-programming-the-right-way/
* [Dispose: Best practice:
  TODO](https://andrewlock.net/four-ways-to-dispose-idisposables-in-asp-net-core/)

* Performance:
  https://docs.microsoft.com/en-us/aspnet/core/performance/performance-best-practices?view=aspnetcore-5.0

* Caching:
  https://docs.microsoft.com/en-us/aspnet/core/performance/caching/response?view=aspnetcore-5.0

* API Versioning:
  https://www.thecodebuzz.com/add-swagger-openapi-api-versioning-net-guidelines/
* API Versioning:
  https://www.infoworld.com/article/3562355/how-to-use-api-versioning-in-aspnet-core.html
* API Versioning + swagger:
  https://dev.to/moesmp/what-every-asp-net-core-web-api-project-needs-part-2-api-versioning-and-swagger-3nfm
  https://github.com/mattfrear/Swashbuckle.AspNetCore.Filters/issues/83
  https://mattfrear.com/2016/01/25/generating-swagger-example-requests-with-swashbuckle/
  https://docs.microsoft.com/en-us/aspnet/core/tutorials/getting-started-with-swashbuckle?view=aspnetcore-5.0&tabs=visual-studio

* Best practice logging

## Worth to know about CancellationTokens



## Worth to know about dotnet api controllers

* Use BaseController instead of Controller Controller: A base class for an MVC
  controller with view support. BaseController: A base class for an MVC
  controller without view support.It is used on Web api in ASP .Net Core. -->
