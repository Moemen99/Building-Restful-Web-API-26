
# Poll Votes API: Returning Raw Vote Data

One of the API endpoints we need is to **retrieve raw vote data** for a specific poll. This will allow the frontend (or any client) to handle custom analytics, charts, or summaries directly without relying on more specialized API endpoints.

---

## 1. Overview of Required Data

When requesting the votes for a given poll (by `pollId`), we want to return:

- **Poll Title**: For context (e.g., “Best Programming Language Poll”).
- **Poll Dates**: (Optional, depending on requirements) - start/end dates, summary, etc.
- **All Votes**: Each vote should include:
  - **Voter Name** (full name or relevant identifier).
  - **Vote Date** (when the user voted).
  - **Selected Answers** (which questions and answers the user chose).

This “raw data” approach makes it easier for the frontend to:

1. Construct any desired figures or charts.  
2. Display detailed summaries or export the data.

---

## 2. Contract Design

Create a new folder (e.g., `Contracts/Results`) and define the records to represent our response model.

### 2.1. `QuestionAnswerResponse`

Represents a **question** and the **answer** chosen by a single user.

```csharp
public record QuestionAnswerResponse(
    string Question,
    string Answer
);
```

### 2.2. `VoteResponse`

Represents **one user’s vote**, including who voted, when they voted, and their selected answers.

```csharp
public record VoteResponse(
    string VoterName,
    DateTime VoteDate,
    IEnumerable<QuestionAnswerResponse> SelectedAnswers
);
```

### 2.3. `PollVotesResponse`

Top-level object for the endpoint’s response. It contains the **poll title** (and potentially other poll metadata) and a list of **votes**.

```csharp
public record PollVotesResponse(
    string Title,
    IEnumerable<VoteResponse> Votes
);
```

> **Note**: You can expand this record to include **start/end dates**, a **summary** field, or any other metadata you need to display or filter by on the frontend.

---

## 3. Next Steps

With these contracts:

1. **Implement the query** in your service/data layer to gather:
   - **Poll Title** (and other poll details).
   - **Votes** with:
     - **User Name** (from joined `User` or `ApplicationUser`).
     - **Vote Date** (from `Vote` entity creation time).
     - **Questions** and **Answers** (join between `VoteAnswers`, `Questions`, and `Answers`).

2. **Create an endpoint** (e.g., `GET /api/polls/{pollId}/raw-votes`) that returns a `PollVotesResponse`.

3. **Map** the database entities to these response contracts:
   - Possibly using **Mapster**, **AutoMapper**, or manual mapping in the service layer.

By structuring the data into these **records**, you provide a **clean** and **easy-to-use** structure for clients who need direct, raw vote data.
```

```
# Returning Poll Votes (Raw Data) via a Results Service

This step illustrates how to implement an endpoint that **retrieves** all votes for a particular poll and returns them in a **raw data format**.  
We’ll create:

1. A **Results Service** (`IResultService` + `ResultService`) that queries the database.  
2. A **ResultsController** that exposes a `GET` endpoint to return the aggregated data.

---

## 1. Creating the `IResultService` and `ResultService`

### 1.1. Interface: `IResultService`

```csharp
public interface IResultService
{
    Task<Result<PollVotesResponse>> GetPollVotesAsync(int pollId, CancellationToken cancellationToken = default);
}
```

This interface defines a single method, `GetPollVotesAsync`, which, given a poll ID, returns either:

- A successful `Result` containing a `PollVotesResponse`, or  
- A failure `Result` (e.g., if the poll doesn’t exist).

---

### 1.2. Implementation: `ResultService`

```csharp
using Microsoft.EntityFrameworkCore;
using SurveyBasket.Data; // Example namespace for ApplicationDbContext

public class ResultService : IResultService
{
    private readonly ApplicationDbContext _context;

    public ResultService(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<Result<PollVotesResponse>> GetPollVotesAsync(
        int pollId,
        CancellationToken cancellationToken = default)
    {
        var pollVotes = await _context.Polls
            .Where(poll => poll.Id == pollId)
            .Select(poll => new PollVotesResponse(
                // 1) Poll Title
                poll.Title,
                // 2) Collection of votes
                poll.Votes.Select(vote => new VoteResponse(
                    // Combine user first/last name
                    $"{vote.User.FirstName} {vote.User.LastName}",
                    // Date/time the vote was submitted
                    vote.SubmittedOn,
                    // Selected answers: question content + answer content
                    vote.VoteAnswers.Select(va => new QuestionAnswerResponse(
                        va.Question.Content,
                        va.Answer.Content
                    ))
                ))
            ))
            .SingleOrDefaultAsync(cancellationToken);

        // If no poll was found, return a failure with an appropriate error
        if (pollVotes is null)
        {
            return Result.Failure<PollVotesResponse>(PollErrors.PollNotFound);
        }

        return Result.Success(pollVotes);
    }
}
```

Key Points:
- We use **EF Core** to query the `Polls` table.
- We join to the associated `Votes` and `VoteAnswers` via **navigation properties**.
- We **project** the results into our `PollVotesResponse` → `VoteResponse` → `QuestionAnswerResponse` chain.

---

### 1.3. Registering the Service in DI

In your **Program.cs** (or **Startup.cs**):

```csharp
services.AddScoped<IResultService, ResultService>();
```

This ensures `ResultService` can be injected wherever `IResultService` is required.

---

## 2. Creating the ResultsController

We now expose an endpoint to fetch the raw vote data for a given poll.

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using YourApp.Extensions; // For result.ToProblem() extension

[Route("api/polls/{pollId}/[controller]")]
[ApiController]
[Authorize]
public class ResultsController : ControllerBase
{
    private readonly IResultService _resultService;

    public ResultsController(IResultService resultService)
    {
        _resultService = resultService;
    }

    [HttpGet("raw-data")]
    public async Task<IActionResult> PollVotes(
        [FromRoute] int pollId,
        CancellationToken cancellationToken)
    {
        var result = await _resultService.GetPollVotesAsync(pollId, cancellationToken);

        return result.IsSuccess
            ? Ok(result.Value)
            : result.ToProblem();
    }
}
```

### Endpoint

- **GET** `api/polls/{pollId}/results/raw-data`  
- **Returns**:  
  - `200 OK` with a `PollVotesResponse` if the poll exists.  
  - A **problem** response (e.g., `404 Not Found`) if the poll ID is invalid or no votes are found.

---

## 3. Example Response

```json
{
  "title": "Poll - 10",
  "votes": [
    {
      "voterName": "Muhammad Ali",
      "voteDate": "2024-05-15T02:25:46.7227923",
      "selectedAnswers": [
        {
          "question": "question_1_updated",
          "answer": "a1"
        },
        {
          "question": "question_2",
          "answer": "a3"
        }
      ]
    }
    // ... additional votes ...
  ]
}
```

### Fields

- **title**: The poll’s title (e.g., “Poll - 10”).
- **votes**: A list of votes. Each vote includes:
  - **voterName**: User’s first + last name.
  - **voteDate**: When the vote was submitted.
  - **selectedAnswers**: Each answer selected, displaying **question** text and **answer** text.

---

## 4. Summary

1. **Data Model**:  
   - `PollVotesResponse` → `VoteResponse` → `QuestionAnswerResponse`.
2. **Service**:  
   - `ResultService.GetPollVotesAsync` queries the database and constructs the nested DTO/record hierarchy.
3. **Controller**:  
   - `ResultsController` exposes the **GET** endpoint, returning `PollVotesResponse` or an error.

With this design, consumers of your API can easily process or display all votes for a given poll, enabling custom UI or data analytics on the **client-side**.
```



```
# Retrieving Votes Per Day for a Specific Poll

Building on our previous **Results** feature, we now add a new endpoint that returns the **number of votes** for a specific poll **grouped by day**. This helps the frontend create charts or statistics about voting trends over time.

---

## 1. New Contract: `VotesPerDayResponse`

In your `Contracts/Results` folder, create a record that represents **one day** and the **number of votes** submitted on that day:

```csharp
public record VotesPerDayResponse(
    DateOnly Date,
    int NumberOfVotes
);
```

- **`Date`**: The day on which votes were cast.  
- **`NumberOfVotes`**: How many votes were cast on that date.

---

## 2. Extending `IResultService`

Add a new method `GetVotesPerDayAsync` to the `IResultService` interface:

```csharp
public interface IResultService
{
    Task<Result<PollVotesResponse>> GetPollVotesAsync(int pollId, CancellationToken cancellationToken = default);
    Task<Result<IEnumerable<VotesPerDayResponse>>> GetVotesPerDayAsync(int pollId, CancellationToken cancellationToken = default);
}
```

---

## 3. Implementing `GetVotesPerDayAsync` in `ResultService`

In your `ResultService`, implement the logic to **group** votes by day and **count** them:

```csharp
public class ResultService : IResultService
{
    private readonly ApplicationDbContext _context;

    public ResultService(ApplicationDbContext context)
    {
        _context = context;
    }

    // Existing method:
    // public async Task<Result<PollVotesResponse>> GetPollVotesAsync(...)

    public async Task<Result<IEnumerable<VotesPerDayResponse>>> GetVotesPerDayAsync(
        int pollId,
        CancellationToken cancellationToken = default)
    {
        // 1. Verify the poll exists
        var pollExists = await _context.Polls
            .AnyAsync(x => x.Id == pollId, cancellationToken);

        if (!pollExists)
        {
            return Result.Failure<IEnumerable<VotesPerDayResponse>>(PollErrors.PollNotFound);
        }

        // 2. Group votes by date and count them
        var votesPerDay = await _context.Votes
            .Where(x => x.PollId == pollId)
            .GroupBy(x => new { Date = DateOnly.FromDateTime(x.SubmittedOn) })
            .Select(g => new VotesPerDayResponse(
                g.Key.Date,
                g.Count()
            ))
            .ToListAsync(cancellationToken);

        return Result.Success<IEnumerable<VotesPerDayResponse>>(votesPerDay);
    }
}
```

### Explanation

1. **Check Poll Existence**: If no poll matches the `pollId`, return a failure result (e.g., `PollErrors.PollNotFound`).  
2. **Group by Date**: We use `DateOnly.FromDateTime(x.SubmittedOn)` to convert the `DateTime` to a `DateOnly`.  
3. **Count Votes**: Each group represents a single day, so we `Count()` the number of votes in that group.

---

## 4. Updating `ResultsController`

Add a new endpoint to return votes per day:

```csharp
[Route("api/polls/{pollId}/[controller]")]
[ApiController]
[Authorize]
public class ResultsController : ControllerBase
{
    private readonly IResultService _resultService;

    public ResultsController(IResultService resultService)
    {
        _resultService = resultService;
    }

    [HttpGet("raw-data")]
    public async Task<IActionResult> PollVotes([FromRoute] int pollId, CancellationToken cancellationToken)
    {
        var result = await _resultService.GetPollVotesAsync(pollId, cancellationToken);
        return result.IsSuccess 
            ? Ok(result.Value) 
            : result.ToProblem();
    }

    [HttpGet("votes-per-day")]
    public async Task<IActionResult> VotesPerDay([FromRoute] int pollId, CancellationToken cancellationToken)
    {
        var result = await _resultService.GetVotesPerDayAsync(pollId, cancellationToken);
        return result.IsSuccess 
            ? Ok(result.Value) 
            : result.ToProblem();
    }
}
```

### Endpoints

1. **`GET /api/polls/{pollId}/results/raw-data`**  
   - Returns detailed vote data (`PollVotesResponse`).

2. **`GET /api/polls/{pollId}/results/votes-per-day`**  
   - Returns an **array** of `VotesPerDayResponse` objects.  
   - Each item includes a **Date** and the **NumberOfVotes** for that day.

---

## 5. Sample JSON Response for `GET /api/polls/{pollId}/results/votes-per-day`

```json
[
  {
    "date": "2024-05-16",
    "numberOfVotes": 5
  },
  {
    "date": "2024-05-17",
    "numberOfVotes": 8
  },
  {
    "date": "2024-05-18",
    "numberOfVotes": 2
  }
]
```

This data can be used by the frontend to plot a **bar chart**, **line chart**, or any other visualization showing the **trend** of votes over time.

---

## Conclusion

With these additions, you now have:

- An endpoint to fetch **raw** vote data (`raw-data`).  
- An endpoint to **aggregate** votes by day (`votes-per-day`).  

These endpoints offer flexibility for frontends or reporting tools to handle analytics and data visualizations without requiring additional custom endpoints.
```


```
# Votes per Question Endpoint

This endpoint returns **each question** for a given poll along with the **answers** that were chosen and how many times each answer was selected. This can help the frontend or reporting systems display percentages or counts of how many users picked a certain answer.

---

## 1. Contracts: `VotesPerQuestionResponse` and `VotesPerAnswerResponse`

In the `Contracts/Results` folder, create two record types to represent the **aggregated** vote data for each question:

```csharp
public record VotesPerQuestionResponse(
    string Question,
    IEnumerable<VotesPerAnswerResponse> SelectedAnswers
);

public record VotesPerAnswerResponse(
    string Answer,
    int Count
);
```

- **`VotesPerQuestionResponse`**:
  - **Question**: The text/content of the question.
  - **SelectedAnswers**: A list of `VotesPerAnswerResponse`, one for each answer that was chosen.

- **`VotesPerAnswerResponse`**:
  - **Answer**: The answer text.
  - **Count**: How many users selected that answer.

---

## 2. Extending the `Question` Entity (Domain Model)

To easily navigate from a question to the votes for that question, ensure the `Question` entity includes a **navigation property** to `VoteAnswer`:

```csharp
public sealed class Question : AuditableEntity
{
    public int Id { get; set; }
    public string Content { get; set; } = string.Empty;
    public int PollId { get; set; }
    public bool IsActive { get; set; } = true;
    
    public Poll Poll { get; set; } = default!;
    public ICollection<Answer> Answers { get; set; } = new List<Answer>();
    
    // Navigation property for VoteAnswer
    public ICollection<VoteAnswer> Votes { get; set; } = new List<VoteAnswer>();
}
```

This allows us to access all votes (`VoteAnswer`) for a specific question.

---

## 3. Service Method: `GetVotesPerQuestionAsync`

In your `IResultService`, define a new method:

```csharp
Task<Result<IEnumerable<VotesPerQuestionResponse>>> GetVotesPerQuestionAsync(
    int pollId,
    CancellationToken cancellationToken = default
);
```

### Implementation in `ResultService`

```csharp
public async Task<Result<IEnumerable<VotesPerQuestionResponse>>> GetVotesPerQuestionAsync(
    int pollId,
    CancellationToken cancellationToken = default)
{
    // 1. Check if the poll exists
    var pollExists = await _context.Polls
        .AnyAsync(p => p.Id == pollId, cancellationToken);

    if (!pollExists)
    {
        return Result.Failure<IEnumerable<VotesPerQuestionResponse>>(PollErrors.PollNotFound);
    }

    // 2. Aggregate votes by question and answer
    var votesPerQuestion = await _context.VoteAnswers
        .Where(va => va.Vote.PollId == pollId)
        .Select(va => new VotesPerQuestionResponse(
            va.Question.Content,
            va.Question.Votes
                .GroupBy(x => new { x.Answer.Id, x.Answer.Content })
                .Select(g => new VotesPerAnswerResponse(
                    g.Key.Content,
                    g.Count()
                ))
        ))
        .ToListAsync(cancellationToken);

    return Result.Success<IEnumerable<VotesPerQuestionResponse>>(votesPerQuestion);
}
```

#### Explanation

1. **Poll Check**: If no poll matches the `pollId`, return a failure result (e.g., `PollErrors.PollNotFound`).  
2. **Query**:
   - Filter by `VoteAnswers` where `Vote.PollId` matches `pollId`.  
   - For each `VoteAnswer`, select a `VotesPerQuestionResponse`:
     - **`Question.Content`**: The question text.  
     - **Group** by answer to count how many times each answer was chosen:
       ```csharp
       va.Question.Votes
         .GroupBy(x => new { x.Answer.Id, x.Answer.Content })
         .Select(g => new VotesPerAnswerResponse(
             g.Key.Content,
             g.Count()
         ))
       ```

---

## 4. Updating `ResultsController`

Add the new endpoint to retrieve votes per question:

```csharp
[Route("api/polls/{pollId}/[controller]")]
[ApiController]
[Authorize]
public class ResultsController : ControllerBase
{
    private readonly IResultService _resultService;

    public ResultsController(IResultService resultService)
    {
        _resultService = resultService;
    }

    [HttpGet("raw-data")]
    public async Task<IActionResult> PollVotes([FromRoute] int pollId, CancellationToken cancellationToken)
    {
        var result = await _resultService.GetPollVotesAsync(pollId, cancellationToken);
        return result.IsSuccess ? Ok(result.Value) : result.ToProblem();
    }

    [HttpGet("votes-per-day")]
    public async Task<IActionResult> VotesPerDay([FromRoute] int pollId, CancellationToken cancellationToken)
    {
        var result = await _resultService.GetVotesPerDayAsync(pollId, cancellationToken);
        return result.IsSuccess ? Ok(result.Value) : result.ToProblem();
    }

    [HttpGet("votes-per-question")]
    public async Task<IActionResult> VotesPerQuestion([FromRoute] int pollId, CancellationToken cancellationToken)
    {
        var result = await _resultService.GetVotesPerQuestionAsync(pollId, cancellationToken);
        return result.IsSuccess ? Ok(result.Value) : result.ToProblem();
    }
}
```

### Endpoint

- **`GET /api/polls/{pollId}/results/votes-per-question`**  
  - Returns a list of `VotesPerQuestionResponse`.  
  - Each item has a question and a list of answers with a **count** of how many times each was chosen.

---

## 5. Example Response

```json
[
  {
    "question": "What is your favorite color?",
    "selectedAnswers": [
      {
        "answer": "Red",
        "count": 10
      },
      {
        "answer": "Blue",
        "count": 15
      },
      {
        "answer": "Green",
        "count": 0
      }
    ]
  },
  {
    "question": "Which programming language do you prefer?",
    "selectedAnswers": [
      {
        "answer": "C#",
        "count": 8
      },
      {
        "answer": "Java",
        "count": 12
      }
    ]
  }
]
```

---

## Summary

With this endpoint, you can retrieve:

1. **Each question** in the poll.
2. **Answers** that were chosen for that question.
3. **Count** of how many votes each answer received.

This data can be used to display **charts**, **percentages**, or any other analysis for how users responded to each question within a specific poll.
```
