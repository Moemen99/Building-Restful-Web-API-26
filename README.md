```
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
