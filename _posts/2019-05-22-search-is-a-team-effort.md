---
typora-copy-images-to: ..\img\posts\search-is-a-team-effort
typora-root-url: ..
layout: post
header_image: /img/posts/search-is-a-team-effort/header.jpg
title: "Search is a Team Effort"
author: Mauro Servienti
synopsis: "Designing search functionality for a distributed system is challenging. There is the coupling threat on the one hand, and on the other we cannot sacrifice efficiency. There are a lot of similarities to what happens during a race car pit stop."
tags:
- soa
- search
category: soa-search
redirect_from: /soa-search/2019/05/22/search-is-a-team-work.html
---

In last installment, [The Quest for a Better Search](<https://milestone.topics.it/soa-search/2019/05/15/the-quest-for-a-better-search.html>), [Tomek](https://twitter.com/Masternak) and myself scratched the surface by looking at the high level problem that searching in a distributed system brings to the table.

Let's use the hotel reservations scenario as a way to identify some requirements:

* Users need to be able to search hotels by location, e.g. a city.
* The search criteria can contain a check-in date and/or a check-out date to filter out unavailable hotels.
* Results can be further filtered by:
  * amenities, such as breakfast included, or a spa. 
  * stars
  * guests reviews and ratings

One can safely assume that the search attributes won't belong to a single service. We won't get into the service boundary identification problem here though, and for the sake of simplicity assume that we are dealing with the following decomposition:

* Registry owns:
  * the hotel location
  * amenities
  * stars
* Reservations owns availability
* Marketing owns guests reviews and ratings

## A naïve design

If we look at the above requirements, in theory, we could design a distributed search process by applying [ViewModel Composition techniques](https://milestone.topics.it/categories/view-model-composition) in the following way:

* Registry will handle the first part of the query returning the list of hotel IDs matching the `location` criteria
* Returned IDs are then handed over to Reservations that filters out IDs based on availability in the selected search period
* Finally IDs are passed to Marketing, and again Registry, to execute a [Faceted search](https://en.wikipedia.org/wiki/Faceted_search) and build facet results.

It works, and is very inefficient. A lot of data will be moved around, across services, for the purpose of filtering them. Both latency and bandwidth will have a huge impact on the overall performance. It would be like if the engineers from the header image of this article were to move the race car to different locations to change tyres or to refuel it.

## A different approach

Let's say the data we want to search are the race car in the header image, and the code owned by each service performing the search is the team of engineers. We want to move the code where data is, and not the other way around. Still, we want to preserve service boundaries as much as we can.

Registry owns a lot of information about hotels other then the aforementioned ones. The identified requirements state that we need to be able to search only by location/city, amenities, and stars. This basically means that Registry could store a document like the following:

```json
{
   'id': 123,
   'city': 'Parma',
   'stars': 4,
   'amenities': [1, 23, 56, 76, 98]
}
```

Reservations could do the same with its reservations data, by storing something similar to:

```json
{
   'hotelId': 123,
   'soldout': '2019-03-12',   
}
```

> We're inventing some data structures here for the sake of the discussion, so don't get too hung up on them; we'll revisit them while looking at a possible implementation. We're using `json` to represent data structures, but the same could be achieved using regular RDBMS tables.

Marketing, using a similar approach, can store ratings and reviews in the same storage.

In essence, every service that needs to participate in the search results processing/generation stores data in the same storage as anyone else, but using different documents.

### Bring what's needed, nothing more, nothing less

The careful reader might have already noticed that we're probably introducing some eventual consistency by using this approach. Both Registry and Reservations need to be sure that searchable data are up-to-date, otherwise they'll produce inconsistent results.

On the one hand we could say that Registry is safe, we don't expect the searchable information to change that often. On the other hand Reservations needs to be very careful. This is the reason the proposed document is as lean as possible, containing the minimum amount of information required to understand if on a given date a certain facility is sold-out.

It's not that different from the team of engineers taking care of the race car during a pit stop. They are bringing to the pit area exactly what they need, nothing more, nothing less. Services do the same with data they "share" into the searchable storage: the more they share the more coupled they are, and the more trouble they might face along the way.

## We need more than data

Data is obviously not enough. Services need to be able to participate in the search process. But that's "easy" at this point, we have many tools in our toolbelt to make it happen. Take a look at the following pseudo-code:

```csharp
SearchResults PerformSearch(Criteria[] criteria, SearchProvider[] searchProviders, FacetsProvider[] facetsProviders)
{
    IQueryable<Hotel> query = searchContext.Hotels;
    foreach(var serachProvider in searchProviders)
    {
        query = searchProvider.AugmentQuery(query, criteria);
    }
    
    var queryResults = query.ToList();
    
    var facets = new List<Facets>();
    foreach(var facetsProvider in facetsProviders)
    {
        facets.Add( facetsProvider.PerformFacetedSearchOver(queryResults) );
    }
    
    return new SearchResults
    {
        Results = queryResults,
        Facets = facets
    };
}
```

For the search part the above snippet uses a syntax similar to the `LINQ/IQueryable` one. The goal is to keep the complexity of the snippet very low. Each service can deploy one or more `searchProvider`s that is injected, thought DI, into the search process. Search providers are given an opportunity to append query statements to the `IQueryable` instance. The Registry search provider could be as simple as:

```csharp
IQueryable<Hotel> AugmentQuery(IQueryable<Hotel> query, Criteria[] criteria)
{
   var locationCriteria = criteria.OfType<LocationCriteria>().Single();
   return query.Where(hotel => hotel.Location == locationCriteria.Value);
}
```

Something very similar happens for facets provider, services will deploy `FacetsProvider` implementations that will be injected and executed during the search process. At this point `SearchResults` will contain the results we need in the most decoupled way, still keeping an eye on efficiency.

There are a lot of improvements that can be done to the very simple search process presented above. That's not the goal of this article though.

## Conclusion

Designing search functionality for a distributed system is indeed challenging. We need to keep an eye on coupling issues, on the one hand, and on the other we don't want to sacrifice efficiency. In this article we've investigated options to design such functionality.

---

Header image: Photo by [Goh Rhy Yan](https://unsplash.com/photos/eo7_DWzUxgw?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/search/photos/f1?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
