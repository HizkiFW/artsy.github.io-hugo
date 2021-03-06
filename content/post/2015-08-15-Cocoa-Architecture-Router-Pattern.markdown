---
author: orta
categories:
- ios
- mobile
- architecture
- energy
- eigen
- eidolon
- hal
comments: false
date: 2015-08-15T00:00:00Z
series: Cocoa Architecture
title: 'Cocoa Architecture: ARRouter'
url: /2015/08/15/Cocoa-Architecture-Router-Pattern/
---

I want to talk about a pattern that we've been using for the last few years on the Artsy Mobile team. This pattern pre-dates me joining Artsy by a few weeks, and was introduced into our codebase by [Ben Jackson](http://90wpm.com), this was the `ARRouter`'s first method:

```objc
  + (NSURL *)newOAuthURLWithUsername:(NSString *)username password:(NSString *)password {
      NSDictionary *params = [[NSDictionary alloc] initWithObjectsAndKeys:
                              username, @"email",
                              password, @"password",
                              ARAuthClientID, @"client_id",
                              ARAuthSecret, @"client_secret",
                              @"credentials", @"grant_type",
                              nil];
      NSString *url_string = [[NSString alloc] initWithFormat:@"%@%@", AROAuthURL, [params queryString]];
      NSURL *url = [ARRouter newURLWithPath:url_string];
      [url_string release];
      [params release];
      return url;
  }
```

Yep, that's pre-ARC, pre-Dictionary Literals, memory-managed code. We took this pattern and rolled with it for the next 4 years, this article is about where we've taken it.

Within Eigen, `ARRouter` is one of our [biggest classes](https://github.com/artsy/eigen/blob/904e8abfc11ce6ea4b6e81f0e02684b755a280c3/Artsy/Networking/ARRouter.m), coming in at almost 1,000 lines of code. Whereas in Energy, it sits at a [more reasonable](https://github.com/artsy/energy/blob/e51529250ede359c781042f222d5836eb9e8a979/Classes/Util/App/ARRouter.m) 300 lines. Eidolon does not have an ARRouter, what gives?

<!--more-->

----------------

## Pattern Evolution

We started out with a Router object as being something that can take a model object, and return a `NSURL` corresponding to a server side end-point.

This worked pretty well, we shipped a 1.0 of Energy with this pattern. However, it become obvious that we were putting a lot of extra knowledge about the type and the parameters of request into classes whose responsibility was not generating a route. For example, user account creation, and user account deletion would use the same `NSURL` but have different HTTP methods.

We migrated our networking stack to using AFNetworking `1.0`, and started using CocoaPods instead of manually dragging and dropping code. With this in mind, we improved on the pattern and started returning `NSURLRequest`s which better encapsulate the server end-point request we were trying to map in the Router.

The pattern evolved when mixed with a [AFHTTPClient](http://cocoadocs.org/docsets/AFNetworking/1.3.4/Classes/AFHTTPClient.html) to act as the base URL resolver, allowing us to easily switch between staging and production environments, and as a central point for hosting all HTTP headers. This meant it was trivial to generate authenticated `NSURLRequest`s.

As it is presently, this pattern is working. We've just wrapped up a new Pod, [Artsy Authentication](https://github.com/artsy/Artsy_Authentication). It's a library that has an `ARRouter` that behaves [exactly like above](https://github.com/artsy/Artsy_Authentication/blob/master/Pod/Classes/ArtsyAuthenticationRouter.h). We continue to build new apps with the pattern.

## Siblings

This pattern is standing the test of time, but that doesn't mean we're not actively trying to experiment within the domain. There are three interesting offshoots from our work on `ARRouter` that are worth talking about.

#### Got the Routes like Swagger

The difference between Eigen's `ARRouter` and Energy's `ARRouter` is pretty simple. Eigen's networking scope is an order of magnitude larger. This is a reflection on the varied data that Eigen is interested in, while Energy has a tight scope on specifically Artsy Parter related data.

During the new year of 2015, I explored the [idea of programmatically generating](https://github.com/orta/GotTheRoutesLikeSwagger) an `ARRouter` as a CocoaPod, and then using CocoaPods' subspecs to make it easy to define what collections of end-points you were interested in. This project is based on a standard in which an API is documented, [Swagger](http://swagger.io). This meant as an API consumer, I can generate the types of `NSURLRequest`s I would require from the API itself. It created files that looked like:

```objc
// Generated by Routes Like Swagger - 31/12/14

@interface ARRouter (User)

/// Retrieve a user by id.
/// @return URLRequest for /api/v1/user/{id}.{format}

- (NSURLRequest *)getUserWithID:(NSString * )slug;

/// Update an existing user.
/// @return URLRequest for /api/v1/user/{id}.{format}

- (NSURLRequest *)updateUserWithID:(NSString * )slug;

... [snip] ...

@end
```

This was a pretty nice expansion of the pattern, but overall felt a bit over-engineered and so, it was left as just an experiment.

#### Moya

When we started an entirely fresh application, we noted down all the networking-related pain points felt from Eigen and Energy. The Router pattern was pretty good, but we were finding that we were having problems with the API consuming part of the `NSURLRequest`s. Mainly, a difficulty in testing, an inconsistency in how we would perform networking and that it didn't feel declarative.

Moya is our attempt at fixing this. I won't go into depth on what Moya is, we've [written articles](/blog/2014/09/22/transparent-prerequisite-network-requests/) on this already. The part that is interesting is that it obviates an ARRouter by using a collection of Swift enums - forcing developers to include all necessary metadata an an end-point.

#### HAL, and API v2

The Router pattern relies on the idea that you know all the routes ahead of time, and add support for them as you build out each part of the app. [HAL, a Hypermedia Application Layer](http://stateless.co/hal_specification.html) - can be approximated as being a self describing API. dB. wrote about it in [this blog post](/blog/2014/09/12/designing-the-public-artsy-api/).

This means that you ask the API how to get certain bits of data, and it will describe the ways in which you can access it.

Artsy's future APIs are using this, and the Router pattern is, more or less, totally deprecated in this world. This is what an artwork's JSON data looks like in v2:

``` json
{
  "id": "4d8b92bb4eb68a1b2c00044a",
  "created_at": "2010-11-15T16:32:38+00:00",
  "updated_at": "2015-08-16T09:26:26+00:00",
  "name": "Jeff Koons",
  "sortable_name": "Koons Jeff",
  "gender": "male",
  "birthday": "1955",
  "hometown": "York, Pennsylvania",
  "location": "New York, New York",
  "nationality": "American",
  "_links": {
    "curies": [
      {
        "name": "image",
        "href": "https://d32dm0rphc51dk.cloudfront.net/Uqad2mGhbNGhAUgb8bUvIA/{rel}",
        "templated": true
      }
  ],
  "thumbnail": {
    "href": "https://d32dm0rphc51dk.cloudfront.net/Uqad2mGhbNGhAUgb8bUvIA/four_thirds.jpg"
  },
  "image:self": {
    "href": "{?image_version}.jpg",
    "templated": true
  },
  "self": {
    "href": "https://api.artsy.net/api/artists/4d8b92bb4eb68a1b2c00044a"
  },
  "permalink": {
    "href": "http://www.artsy.net/artist/jeff-koons"
  },
  "artworks": {
    "href": "https://api.artsy.net/api/artworks?artist_id=4d8b92bb4eb68a1b2c00044a"
  },
  "published_artworks": {
    "href": "https://api.artsy.net/api/artworks?artist_id=4d8b92bb4eb68a1b2c00044a&published=true"
  },
  "similar_artists": {
    "href": "https://api.artsy.net/api/artists?similar_to_artist_id=4d8b92bb4eb68a1b2c00044a"
  },
  "similar_contemporary_artists": {
    "href": "https://api.artsy.net/api/artists?similar_to_artist_id=4d8b92bb4eb68a1b2c00044a&similarity_type=contemporary"
  },
  "genes": {
    "href": "https://api.artsy.net/api/genes?artist_id=4d8b92bb4eb68a1b2c00044a"
  }
  },
  "image_versions": [
    "four_thirds",
    "large",
    "square",
    "tall"
  ]
}
```

You can see that via the _links section, curies and self-referential urls, you can build network client which traverses the API without built-in implicit knowledge.

It's a really exciting pattern, and as client developers, we can work on improving standard API clients that work on all HAL APIs. Instead of something specific to Artsy's API. A lot of the most interesting work in the Cocoa space has been done by Kyle Fuller with [Hyperdrive](https://cocoapods.org/pods/Hyperdrive).

### Wrap Up

Given that we're not writing applications against the v2 API, yet. The Router pattern is working fine for us at Artsy. It can be a really nice way to abstract out a responsibility that may currently be sitting inside a very large API client that might be worth extracting out.

Let us know what you think, send tweets to [@ArtsyOpenSource](https://twitter.com/ArtsyOpenSource) on twitter. Ps. it's pronounced "rooter".
