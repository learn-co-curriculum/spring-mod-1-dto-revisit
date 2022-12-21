# DTOs Revisited

## Learning Goals

- Learn about the `@JsonProperty` annotation.
- Discuss the use case of hiding and revealing information in data transfer
  objects.
- Explain the difference between a data transfer object and an entity.

## Review of Data Transfer Objects

As a reminder, a **data transfer object (DTO)** is an object that encapsulates
data in order to carry it between processes. This is a design pattern that
reduces the amount of calls and processes back and forth to the server by
batching up multiple parameters in a single call.

As we saw before, we didn't even need to create a serializer to serialize or
deserialize the objects to go from a JSON to a DTO and vice versa. The other
advantages to utilizing DTOs is that we can customize the JSON properties by
using annotations like `@JsonProperty` along with hiding certain fields that we
may not want to reveal when passing data back to the client.

In this lesson, we'll cover more of the different things we can do with DTOs and
then explain the differences between a DTO and an entity class.

## Java Variable Names and JSON Property Names

Consider our football team example again:

```java
package com.example.springdatademo.dto;

import lombok.Data;

@Data
public class FootballTeamDTO {
    private String teamName;
    private int wins;
    private int losses;
    private boolean currentSuperBowlChampion;
}
```

When we perform a GET request currently, we may get a JSON response back that
looks like this:

```json
{
    "teamName":"Dallas-Cowboys", 
    "wins":7, 
    "losses":3, 
    "currentSuperBowlChampion":0
}
```

Now what if we wanted `teamName` and `currentSuperBowlChamption` to be
separated by underscores instead of using the camel case convention?

We could simply change our variable names to look like this...

```java
package com.example.springdatademo.dto;

import lombok.Data;

@Data
public class FootballTeamDTO {
    private String team_name;
    private int wins;
    private int losses;
    private boolean current_super_bowl_champion;
}
```

But that isn't the standard way of defining variables in Java. Instead, we can
specify the property name in a JSON by using the Jackson annotation,
`@JsonProperty`.

```java
package com.example.springdatademo.dto;

import com.fasterxml.jackson.annotation.JsonProperty;
import lombok.Data;

@Data
public class FootballTeamDTO {

    @JsonProperty("team_name")
    private String teamName;

    private int wins;

    private int losses;

    @JsonProperty("current_super_bowl_champion")
    private boolean currentSuperBowlChampion;
}
```

Now if we send a GET request to our API, we might get back a JSON file like
this:

```json
{
    "wins": 7,
    "losses": 3,
    "team_name": "Dallas-Cowboys",
    "current_super_bowl_champion": false
}
```

### JSON Ordering of Properties

We might notice in the JSON above that the order of the properties changed when
we specified what the name should be in the JSON in the `FootballTeamDTO` class.

If we want a specific ordering, like perhaps we do want the JSON to be in the
order of: team name, wins, losses, current super bowl champion, then we can
use another Jackson annotation called `@JsonPropertyOrder` at the class level.

```java
package com.example.springdatademo.dto;

import com.fasterxml.jackson.annotation.JsonProperty;
import com.fasterxml.jackson.annotation.JsonPropertyOrder;
import lombok.Data;

@Data
@JsonPropertyOrder({"team_name", "wins", "losses", "current_super_bowl_champion"})
public class FootballTeamDTO {
    
    @JsonProperty("team_name")
    private String teamName;

    private int wins;

    private int losses;

    @JsonProperty("current_super_bowl_champion")
    private boolean currentSuperBowlChampion;
}
```

The `@JsonPropertyOrder` will ensure that the JSON is returned to the client in
the order we specified: team_name, wins, losses, current_super_bowl_champion:

```json
{
    "team_name": "Dallas-Cowboys",
    "wins": 7,
    "losses": 3,
    "current_super_bowl_champion": false
}
```

## Hiding and Revealing Information in DTOs

Consider the case where we might want to hide data in a JSON response. The most
common case is when we have a `password` field. But for consistency, let's
assume for a minute that we don't want to return the `currentSuperBowlChampion`
field in the JSON response body.

There are a few things we could do. We could either:

- Remove the getter and setter methods for the field.
- Add a Jackson annotation, `@JsonIgnore`.
- Add a Jackson annotation, `@JsonProperty`.

If we remove the getter and setter, then when we try to perform a GET request,
it will not be able to perform a GET on the `currentSuperbowlChampion` field.
Therefore, the response would look like this:

![Get-Football-No-Superbowl](https://curriculum-content.s3.amazonaws.com/spring-mod-1/dto/json-without-superbowl-boolean.png)

But by also removing the setter, we wouldn't be able to properly POST a football
team since there is no method to set the `currentSuperbowlChampion`. If we still
want to set the field, then we'd have to leave the setter in the code.

A better way to go about this may be to use the `@JsonIgnore` and
`@JsonProperty` annotations that are part of the
`com.fasterxml.jackson.annotation` package.

The `@JsonIgnore` annotation tells Jackson to completely ignore this field when
it comes to serializing and deserializing the JSON. This is essentially the same
as removing both the getter and setter of a private instance variable if this
annotation is applied at the field level:

```java
        @JsonIgnore
        @JsonProperty("current_super_bowl_champion")
        private boolean currentSuperBowlChampion;
```

But in the case we mentioned above, maybe we want to be able to set this field,
just not return this field when we perform a GET request. In that case, we can
use the `@JsonProperty` annotation again! We'll modify the annotation to look
like this:

```java
        @JsonProperty(value = "current_super_bowl_champion", access = JsonProperty.Access.WRITE_ONLY)
        private boolean currentSuperBowlChampion;
```

What this annotation is doing is telling Jackson to only deserialize the field.
Therefore, it can write out to the field, `currentSuperBowlChampion`, but it
will not show up when a GET request is performed. This is essentially the same
as removing the getter of a private instance variable if this annotation is
applied at the field level.

### The Use Case For Two DTOs

Now let us consider another scenario. Say we have two GET request methods in our
`FootballController`:

```java
    /**
     * Get a football team by the team name
     * @param teamName : String - name of the football team of interest
     * @return FootballTeamDTO
     */
    @GetMapping("/football-team/{teamName}")
    public FootballTeamDTO getFootballTeam(@PathVariable String teamName) {
        return footballService.getFootballTeam(teamName);
    }

    /**
     * Get all the football teams in the data source
     * @return List<FootballTeamDTO>
     */
    @GetMapping("/football-teams")
    public List<FootballTeamDTO> getFootballTeams() {
        return footballService.getAllFootballTeams();
    }
```

And in these two GET requests, let's say we have a requirement to return a JSON
in the following format when we send the GET request URL
"http://localhost:8080/football-team/Dallas-Cowboys":

```json
{
    "team_name": "Dallas-Cowboys",
    "wins": 7,
    "losses": 3,
    "current_super_bowl_champion": 0
}
```

But we also have a requirement to return a JSON in the following format when we
send the other GET request "http://localhost:8080/football-teams":

```json
[
    {
        "team_name": "Dallas-Cowboys",
        "wins": 7,
        "losses": 3
    },
    {
        "team_name": "Pittsburgh-Steelers",
        "wins": 4,
        "losses": 7
    }
]
```

Notice that the one JSON would show the `currentSuperBowlChampion` field, but
the other request does not.

Let's look at our current `FootballTeamDTO` class again:

```java
package com.example.springdatademo.dto;

import com.fasterxml.jackson.annotation.JsonProperty;
import com.fasterxml.jackson.annotation.JsonPropertyOrder;
import lombok.Data;

@Data
@JsonPropertyOrder({"team_name", "wins", "losses", "current_super_bowl_champion"})
public class FootballTeamDTO {
    
    @JsonProperty("team_name")
    private String teamName;

    private int wins;

    private int losses;

    @JsonProperty(value = "current_super_bowl_champion", access = JsonProperty.Access.WRITE_ONLY)
    private boolean currentSuperBowlChampion;
}
```

Assume for a minute that the service code has already been written to retrieve
all the football teams from the data source. If we were to run the application
as is, we could match the JSON format for the GET-all request:
`http://localhost:8080/football-teams`.

However, if we send the GET request with a specific team as the path variable,
`http://localhost:8080/football-team/Dallas-Cowboys`, we'd get a JSON that
excludes the `currentSuperBowlChampion` field still:

```json
{
    "team_name": "Dallas-Cowboys",
    "wins": 7,
    "losses": 3
}
```

In this scenario, we might try to remove the
`access = JsonProperty.Access.WRITE_ONLY` from the `@JsonProperty` on the
`currentSuperBowlChampion` field. If we do this, then we'd match the GET request
JSON format where we specify a certain team, but would mismatch the JSON format
when performing a GET-all request.

So what do we do from here?? How do we get them _both_ to match?

This is a use case for creating a separate DTO, despite only having one entity
class, in order to meet the serialization requirements that are expected.

Let's start by creating a DTO that **includes** the `currentSuperBowlChampion`
field. We'll call this class `FootballTeamWithChampionDTO`:

```java
package com.example.springdatademo.dto;

import com.fasterxml.jackson.annotation.JsonProperty;
import com.fasterxml.jackson.annotation.JsonPropertyOrder;
import lombok.Data;

@Data
@JsonPropertyOrder({"team_name", "wins", "losses", "current_super_bowl_champion"})
public class FootballTeamWithChampionDTO {

    @JsonProperty("team_name")
    private String teamName;

    private int wins;

    private int losses;

    @JsonProperty("current_super_bowl_champion")
    private boolean currentSuperBowlChampion;
}
```

Next, we'll create a DTO that **excludes** the `currentSuperBowlChampion` field.
Name this class `FootballTeamNoChampionDTO`:

```java
package com.example.springdatademo.dto;

import com.fasterxml.jackson.annotation.JsonProperty;
import com.fasterxml.jackson.annotation.JsonPropertyOrder;
import lombok.Data;

@Data
@JsonPropertyOrder({"team_name", "wins", "losses"})
public class FootballTeamNoChampionDTO {

    @JsonProperty("team_name")
    private String teamName;

    private int wins;

    private int losses;
}
```

Now in the service and controller classes, we'll make the modifications to
return the appropriate DTO object:

```java
// FootballService.java (GET methods only)

    // !! Notice this will return a FootballTeam that includes the currentSuperBowlChampion field
    public FootballTeamWithChampionDTO getFootballTeam(String teamName) {
        Optional<FootballTeam> optionalFootballTeam = footballRepository.findFootballTeamByTeamName(teamName);
        FootballTeam footballTeamEntity = optionalFootballTeam.orElseThrow();
        
        // Have the ModelMapper map to the FootballTeamWithChampionDTO
        return modelMapper.map(footballTeamEntity, FootballTeamWithChampionDTO.class);
    }

    // !! Notice this will return a List of FootballTeams that excludes the currentSuperBowlChampion field
    public List<FootballTeamNoChampionDTO> getAllFootballTeams() {
        List<FootballTeamNoChampionDTO> footballTeamDTOS = new ArrayList<>();
        Iterable<FootballTeam> footballTeams = footballRepository.findAll();
        for (FootballTeam footballTeam : footballTeams) {
            
            // Have the ModelMapper map to the FootballTeamNoChampionDTO
            FootballTeamNoChampionDTO footballTeamDTO = modelMapper.map(footballTeam, FootballTeamNoChampionDTO.class);
            footballTeamDTOS.add(footballTeamDTO);
        }

        return footballTeamDTOS;
    }
```

```java
// FootballController.java (GET methods only)

    /**
     * Get a football team by the team name
     * @param teamName : String - name of the football team of interest
     * @return FootballTeamWithChampionDTO - will include if the team is the current super bowl champion
     */
    @GetMapping("/football-team/{teamName}")
    public FootballTeamWithChampionDTO getFootballTeam(@PathVariable String teamName) {
        return footballService.getFootballTeam(teamName);
    }

    /**
     * Get all the football teams in the data source
     * @return List<FootballTeamNoChampionDTO> - will exclude if the team is the current super bowl champion
     */
    @GetMapping("/football-teams")
    public List<FootballTeamNoChampionDTO> getFootballTeams() {
        return footballService.getAllFootballTeams();
    }
```

If we run the application now with these changes, we'll notice that we get the
expected JSON formats that were specified in the requirements!

In this particular case, it was beneficial to create two separate DTO objects
to properly serialize the data for the different expected responses.

Note: This is one way to create the DTO objects; however, one could make use of
inheritance by having the DTO with the `currentSuperBowlChampion` field inherit
from the DTO that excluded the field. This would reduce repetitive lines of
code.

## DTO versus Entity

Within this lesson thus far, we might now be able to tell the differences
between a DTO and an entity. A DTO is used **only** to transfer data from one
process to another. Possibly from an API to a client.

| DTO                                                        | Entity                                               |
|------------------------------------------------------------|------------------------------------------------------|
| Used **only** to transfer data from one process to another | Typically used in the context with JPA functionality |
| A POJO that can be serialized into a specific JSON format  | An object that represents a database table           |

## Conclusion

In this lesson, we learned about how powerful a DTO can be when we want to
serialize data in a certain way and hide and reveal information when sending
DTOs from one process to another. We also learned how a DTO differs from an
entity class and further our understanding of DTOs.

## Resources

- [Using @JsonIgnore or @JsonProperty](https://medium.com/@bhanuchaddha/using-jsonignore-or-jsonproperty-to-hide-sensitive-data-in-json-response-ad12b1aacbf3)
- [Baeldung Java DTO Pattern](https://www.baeldung.com/java-dto-pattern)
- [Baeldung Jackson Annotations](https://www.baeldung.com/jackson-annotations)
- [Difference Between Entity and DTO](https://www.linkedin.com/pulse/difference-between-entity-dto-what-use-instead-omar-ismail?trk=pulse-article_more-articles_related-content-card)
