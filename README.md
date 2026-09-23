# Halfsies

**Meet in the middle — fairly.**

Halfsies is an app where two people enter their starting locations and get back restaurants and activities that take *roughly the same travel time* for both of them to reach.

Most "meet halfway" tools pick the geographic midpoint. That's rarely fair: a point that's 5 miles from each of you might be a 10-minute drive for one person and a 45-minute transit slog for the other. Halfsies optimizes for **time**, not distance.

## How it works

1. **Enter two locations.** Addresses, neighborhoods, or current location.
2. **Pick how each person is traveling.** Driving, transit, walking, or cycling — each person can choose their own mode.
3. **Choose what you're looking for.** Restaurants, coffee, bars, parks, museums, or other activities.
4. **Get fair results.** Halfsies ranks places by how balanced the travel times are, and shows each person's estimated trip time side by side.

## Features (planned)

- [ ] Two-origin input with address autocomplete
- [ ] Per-person travel mode (drive / transit / walk / bike)
- [ ] Travel-time-balanced search using isochrones or travel-time matrices
- [ ] Fairness ranking: minimize the difference in travel time, then total travel time
- [ ] Category filters (food, drinks, coffee, outdoors, entertainment)
- [ ] Filters for rating, price level, and "open now"
- [ ] Map view showing both routes and the candidate spots
- [ ] Departure-time awareness (traffic and transit schedules vary by time of day)
- [ ] Shareable link so the other person can see the options

## Approach

At a high level:

1. Compute a travel-time **isochrone** (reachable area within *N* minutes) for each person using their chosen travel mode.
2. Find the overlap region where both people can arrive within similar time budgets, expanding *N* until there are enough candidates.
3. Query places APIs for venues in that region matching the chosen category.
4. Fetch actual travel times from each origin to each candidate via a distance/travel-time matrix API.
5. Score each candidate, for example:

   ```
   score = |t_A - t_B| + λ · (t_A + t_B)
   ```

   where `t_A` and `t_B` are each person's travel time, and `λ` controls how much total travel time matters compared to fairness.

6. Return the top results, sorted by score.

### Candidate data sources

| Need | Options |
| --- | --- |
| Geocoding / autocomplete | Google Places, Mapbox, OpenStreetMap Nominatim |
| Isochrones | Mapbox Isochrone API, OpenRouteService, TravelTime API, Valhalla |
| Travel-time matrix | Google Distance Matrix / Routes API, Mapbox Matrix, OSRM |
| Places / venues | Google Places, Foursquare, Yelp Fusion, OpenStreetMap (Overpass) |

## Tech stack

Native mobile apps for iOS and Android, backed by the Halfsies API, which proxies map and places API calls so API keys aren't exposed to clients. There is no web client in v1. The app framework is still to be decided (see `REQUIREMENTS.md` OD-01).

## Privacy

Halfsies handles location data, so the defaults should be conservative:

- Don't store origin locations after a search completes, unless the user saves them
- Keep API keys server-side
- Shared links carry results, not either person's exact starting address

## Contributing

The project is at an early stage. Issues and ideas are welcome — open an issue to discuss before sending a large PR.

## License

[MIT](LICENSE)
