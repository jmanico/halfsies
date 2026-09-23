<p align="center"><img src="logo.svg" alt="Halfsies" width="320"></p>

# Halfsies

**Meet in the middle — fairly.**

Halfsies is a mobile app for iOS and Android. Two people each enter their starting point and get back restaurants and activities that take *roughly the same travel time* for both of them to reach.

Most "meet halfway" tools pick the geographic midpoint. That's rarely fair: a point that's 5 miles from each of you might be a 10-minute drive for one person and a 45-minute transit slog for the other. Halfsies optimizes for **time**, not distance.

## How it works

1. **Start a session.** One person signs in (Sign in with Apple or Google) and sends an invite link through the phone's share sheet.
2. **Join.** The other person opens the link and joins, as a guest if they like.
3. **Set your own starting point.** Each person uses address search, their current location, or a pin on the map, and picks their own travel mode: drive, transit, walk, or bike.
4. **Get fair results.** Halfsies finds places where both travel times are close and shows both times side by side. When the times are close enough, the place gets an **Even trip** badge.
5. **Agree on a plan.** Either person proposes a place, the other accepts, and both open directions in Apple Maps or Google Maps.

## Scope (v1)

- Native mobile apps for iOS and Android, backed by the Halfsies API
- Exactly two people per session
- Categories: restaurants, cafes, bars, and activities (parks, museums, entertainment)
- Filters: category, open at meeting time, price level, maximum travel time per person
- Meeting time: now, or up to 14 days ahead
- Push notifications for session events

There is no web client, no groups of more than two, and no reservations, payments, chat, or ads in v1. See [`REQUIREMENTS.md`](REQUIREMENTS.md) for the full list.

## Approach

1. Find the area both people can reach, using each person's own travel mode and the meeting time. The search does not rely on the geographic midpoint alone.
2. Search that area for places that match the filters.
3. Get real travel times from each starting point to each place (`tA`, `tB`) from a travel-time matrix.
4. Rank the places (lower score is better):

   ```
   score = max(tA, tB) + W · |tA - tB|
   ```

5. Mark a place as an **Even trip** when `|tA - tB| <= max(0.15 · max(tA, tB), 300 s)`.

The weight `W` and the Even trip thresholds are server-side settings (FR-SRCH-06, FR-SRCH-07).

### Candidate providers

| Need | Candidates |
| --- | --- |
| Travel-time matrix (drive, transit, walk, bike) | Google Routes API, Mapbox Matrix API, self-hosted Valhalla or OSRM (transit requires a GTFS-capable engine such as OpenTripPlanner) |
| Places (category, hours, price) | Google Places API, Foursquare Places API |
| Push | APNs, Firebase Cloud Messaging |
| Identity | Sign in with Apple, Google (OIDC) |

The final provider choices are open decisions (OD-02, OD-03).

## Tech stack

Native mobile apps for iOS and Android, backed by the Halfsies API. The API is the only backend the apps talk to. It proxies all place and routing calls, so provider API keys never ship in the apps. The app framework is still to be decided (OD-01).

## Privacy

Halfsies handles location data, so the defaults are conservative:

- Neither person ever sees the other's exact starting point. They see only a general area label.
- Location is requested only when you tap "Use my current location", never in the background.
- Exact starting points are encrypted and deleted when the session ends.
- Invite links and notifications never contain anyone's location.
- There are no ad or tracking SDKs.

## Docs

- [`REQUIREMENTS.md`](REQUIREMENTS.md): product, API, privacy, and security requirements
- [`DESIGN.md`](DESIGN.md): visual design language
- [`style-guide.html`](style-guide.html): reference rendering of the design tokens

## Contributing

The project is at an early stage. Issues and ideas are welcome. Please open an issue to discuss before sending a large PR.

## License

[MIT](LICENSE)
