I'd like to build an animated football visualisation that looks like the wind maps on weather
websites, but shows how a team moves the ball instead of how the wind blows.

The idea: every possession is a route — it starts somewhere, travels through a chain of passes and
carries, and ends somewhere. Lay every possession a team had across a season onto one pitch and
their habits appear as currents. I want those currents drawn as a flowing particle field: routes
the team uses a lot become thick, bright channels; areas they rarely use stay calm and empty; and
particles move faster where the ball actually moves faster (work out ball speed from the event
timestamps), so a quick switch looks different from patient build-up.

It's a comparison tool, so two pitches side by side, each with its own team dropdown, to see at a
glance how two teams progress the ball differently.

Data: StatsBomb open data, FA Women's Super League 2023/24 (every team has 22 games). Use only the
team in possession's own passes and carries, leave out corners and kick-offs, and cache the
downloads so reruns are quick. Python for the data prep, plain HTML/JS canvas for the front end,
no frameworks.

The thing I most want to avoid is a hairball — lines going everywhere in every direction until
nothing stands out. In my experience that comes from a few things, so please design against them:
- Don't average directions in an area. If a team goes left half the time and right the other half
  from the same spot, the average cancels out and hides both. Keep track of which way the ball goes
  from each area, then only draw the dominant direction (or two, if it's a genuine even split).
- Long passes shouldn't swamp an area just because they're long; count how much ball travel actually
  passes through each spot.
- Be strict about what counts as a route. Areas with only light traffic should go completely calm
  rather than showing faint noise, and particles should cluster heavily on the busiest routes rather
  than spreading evenly. Judge "busy" against the whole league, so both pitches share one scale and
  a thick line means the same on either side.
- Keep channels narrow — light smoothing, and let particles drift towards the core of the channel
  they're in so the flow forms tight rivers rather than broad smears.
- Particles should keep following their own current, not jitter between neighbouring ones.
- Ball speeds end up quite similar across the pitch once averaged, so exaggerate the differences in
  particle speed and colour, otherwise everything moves and looks the same.
- Make sure fading trails genuinely disappear — canvas fades can leave a permanent haze behind.

Pitch: dark background, crisp white markings including the penalty-area arcs and spots. Both teams
attack left to right.

Controls: keep it simple. One shared row of three sliders affecting both pitches — Detail (how much
secondary flow shows), Channel width, and Trail length — and fix everything else at sensible
values. Add a small legend for the speed colours and what line thickness means, and stack the
pitches on narrow screens.
