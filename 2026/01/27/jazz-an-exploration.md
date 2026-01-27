# Jazz: An Exploration

<script src="./jazz-components.min.js"></script>

For the past few years, I've been interested in jazz. I want to learn more about
the genre. I don't have a good ear, and I have zero musical training, which
makes this difficult.

I finally got around to reading Ted Gioia's [The History of
Jazz](https://www.goodreads.com/book/show/177539.The_History_of_Jazz).
This is an incredibly well-researched book by someone who obviously loves the
genre. I've been enjoying it, though I'm not making the most out of it myself.
The text casually mentions songs everywhere, which I need to search and listen
to. I sometimes do this, sometimes, especially if my phone isn't handy, I don't.
Many of the passages refer to music theoretical terms that I do not grasp. I
realized I need a slightly different medium to improve my study.

I ended up building [Jazz: An Exploration](https://vladris.com/jazz-book/).

## Something between a book and a wiki

A book seems to be too linear for a topic like jazz. I don't want to break the
flow to find a song. I want a way to quickly reference a term like *chord* if
I don't understand it.

A wiki doesn't have a narrative thread. I can jump from article to article
though it's hard to distinguish between progress and going down some side rabbit
hole.

To counter my own limitations, I came up with something that sits between a book
and a wiki. The two main elements I've been exploring are *threads* and
*interactive elements*.

### Threads

I view jazz as best explored through multiple lines. The backbone is the
history: where it originated, how it evolved. The story of the genre. A parallel
thread is the key players and innovators. For example, Miles Davis was a key
figure in cool jazz, modal jazz, fusion etc. His biography runs parallel to the
different historical eras. I call these parallel lines *threads*.

My exploration of jazz has a *history* thread, and a parallel *players* thread.
When the history mentions a key figure, you can switch threads to read about
the player. Or, vice versa, reading about the players in chronological order,
you can hop threads to read about the history of that era.

All in all I included five threads in the exploration:

* History: From origin until after fusion.
* Players: Key players that influenced the genre.
* Recordings: Genre-defining recordings, including listening guides.
* Standards: A selection of jazz standards, with recordings by different artists
  and listening guides.
* Theory: Some music theory basics to help decode jazz.

### Interactive elements

The interactive elements idea is nothing new, but I personally find it very
useful. The text doesn't just mention a song, there's an embedded Apple Music
player so listening to it is one click away:

<div id="player"></div>
<script>JazzComponents.renderAppleMusic(document.getElementById('player'), {
    id: "1249030790?i=1249031216",
    title: "Black Lace Freudian Slip",
    artist: "René Marie",
    country: "us"
});</script>

For the music theory part, I included several interactive components.

Here's musical notation that can be played:

<div id="notation"></div>
<script>JazzComponents.renderNotation(document.getElementById('notation'), {
    notes: "C4/8 C#4/8 D4/8 D#4/8 E4/8 F4/8 F#4/8 G4/8",
    clef: "treble",
    time: "4/4",
    tempo: 140
});</script>

Here's an interactive piano (multi-touch supported) for playing scales:

<div id="piano"></div>
<script>JazzComponents.renderPiano(document.getElementById('piano'), {
    root: "C",
    mode: "major",
    octaves: 2,
    startOctave: 4,
    interactive: true
});</script>

Here's a chord player:

<div id="progression"></div>
<script>JazzComponents.renderProgression(document.getElementById('progression'), {
    chords: "Dm7|G7|Cmaj7",
    tempo: 100,
    beats: 4,
    loop: false,
    label: "ii-V-I in C Major"
});</script>

And a drum pattern to explain rhythm:

<div id="drums"></div>
<script>JazzComponents.renderDrums(document.getElementById('drums'), {
    pattern: "swing",
    tempo: 120
});</script>

The main theme here is to seamlessly transition between reading and listening.

## Implementation

I used AI both to implement this and to generate the content. As a writer, I am
very much opposed to AI generating content. I would never use it and claim
authorship. I firmly believe a writer needs to use their own voice. Disclaimer
aside, in this particular case is something I'm doing for my own learning. I
know so little about jazz that it would be impossible for me to produce the
content. I'm leveraging AI as a learning tool.

I have two interesting anecdotes from putting together this project: embedding
the music player for the various songs, and generating content.

### Music player

I originally tried to embed the Spotify player. Both Spotify and Apple Music
embeds rely on a track ID which uniquely identifies the song. Claude happily
hallucinated IDs for all songs mentioned. 5 of them worked out of around 250
songs referenced.

I tried a couple of approaches to fix this, since looking up 250 songs by hand
would take quite a while. First, I let Claude access Spotify which promptly got
us rate-limited. After a cooldown period, once I was able to access Spotify
again, I tried getting an API key to access their endpoint programmatically.
Unfortunately it seems like at the moment, Spotify wasn't allowing the creation
of new apps.

I decided to switch to Apple Music. This went a lot smoother, the public
endpoint Apple exposes is not as restrictive as Spotify's. Claude was able to
search for each individual song by artist and title and get the correct IDs
without getting rate-limited.

One issue I am aware of is it didn't do a good job when we needed a particular
recording. Jazz artists might record the same song many times. For example, a
landmark recording is Benny Goodman's Carnegie Hall recording in 1938. The
songs found on Apple Music are by the same artist and have the same title, but
they're not the ones recorded live at Carnegie Hall.

This issue is something that I could address in probably several ways: just
manually find the right song since the list of songs where the specific
recording matters is quite small; alternately guide Claude to better navigate
Apple Music. For now, I can live with it - I suspect my ear is not trained
enough to pick up on differences between recordings.

### Content generation

The other interesting challenge was generating content. This was a multi-step
process. First, I started with the very high-level: What is the list of key
players we should cover? Next, for each one, generate a high-level summary,
the bullet points we need to cover. Finally, flesh out the chapter.

This isn't enough though! What prevents the model from hallucinating facts? I
ended up building a validation pipeline. For reach article, I had Claude extract
*claims* in the following format:

```json
{
  "version": "1.0",
  "sourceFile": "content/players/21-miles-davis.md",
  "extractedAt": "2026-01-24T12:00:00Z",
  "claims": [
    {
      "id": "players-21-miles-davis-L15-1",
      "type": "biographical",
      "subtype": "birth",
      "text": "Miles Dewey Davis III was born on May 26, 1926, in Alton, Illinois",
      "line": 15,
      "entities": {
        "names": ["Miles Dewey Davis III"],
        "dates": ["1926-05-26"],
        "locations": ["Alton, Illinois"]
      },
      "verificationHint": "Miles Davis birth date birthplace"
    },
    {
      "id": "players-21-miles-davis-L42-2",
      "type": "recording",
      "subtype": "session",
      "text": "Kind of Blue was recorded on March 2 and April 22, 1959, at Columbia's 30th Street Studio",
      "line": 42,
      "entities": {
        "names": ["Miles Davis"],
        "dates": ["1959-03-02", "1959-04-22"],
        "locations": ["Columbia's 30th Street Studio", "New York"],
        "works": ["Kind of Blue"]
      },
      "verificationHint": "Kind of Blue recording dates studio"
    }
  ]
}
```

I doubt the format itself matters much, this is what Claude itself suggested.
The key point being to extract this information.

Next, I had it cross-check this with Wikipedia and other web sources. In this
iteration, it read a claims file and, claim by claim, made sure there is
information to back it on the web. The prompt specifically insisted on not
relying on internal knowledge for the validation. Here's a snippet:

```markdown
You are validating factual claims extracted from educational content about jazz
history. For each claim, you must verify it against external authoritative
sources and attach the reference. **Do NOT rely on your training data—all
validation must come from fetched external sources.**

### Why External Sources Only?

This validation process exists specifically to catch hallucinations and errors.
If we allowed the model to validate claims from its own knowledge, we'd be
validating potentially hallucinated content against potentially hallucinated
knowledge. Every claim must have a fetched, citable source.

### Authoritative Sources (in priority order)

| Source | Best For | How to Access |
|--------|----------|---------------|
| **MusicBrainz** | Recording dates, personnel, releases, labels | `fetch_webpage` on musicbrainz.org URLs |
| **Wikidata** | Structured data: birth/death dates, places, identifiers | `fetch_webpage` on wikidata.org URLs |
| **Wikipedia** | Biographical details, historical context, career facts | `fetch_webpage` on wikipedia.org URLs |
| **AllMusic** | Album details, session info, personnel | `fetch_webpage` on allmusic.com URLs |
| **Discogs** | Release dates, pressings, personnel | `fetch_webpage` on discogs.com URLs |
| **Web Search** | Fallback for claims not found in above sources | Use search to find authoritative sources |
```

Running validation over the extracted claims identified several inaccurate
"facts." Somewhere between 20-30% of claims were false. Most of these were not
egregious though - mostly dates mixed up, off by a year etc. A much small number
ended up being unverifiable.

Is this a foolproof method? I doubt it. I expect the model to have missed
some claims during extraction and I noticed the quality of research to vary
a lot. In some cases, it simply checked Wikipedia and if it couldn't find
something it said it is unverifiable. In other instances it did a lot more
digging, searching the web, and so on.

I expect accuracy to increase with multiple iterations of this: rather than a
single claim extraction pass, repeat claim extraction several times to get
better coverage. Similarly, during claim validation, attempting to validate the
same claim multiple times should yield better accuracy.

## Thoughts

Coding models are pretty great these days. I stood up the whole project over
a single weekend. Building an interactive study guide from scratch in two days
(including the content!) is quite amazing.

I wonder what this means for learning. It's going to be extremely easy to have
personalized study material generated on-demand.

I've been enjoying the multi-threaded reading experience. Claude named it
"aspect-oriented learning," which fits very well.

A good next step when I have some free time is to cleanup the code I vibed and
open-source it. It's pretty messy at the moment, but I envision something like:

* A compiler that takes Markdown sources.
* A plugin system for components - The musical components I used for my Jazz
  exploration being an instance of this. I can imagine many other possible
  component bundles like charts and graphs, math, code etc.
* A configurable app that can serve the compiled threads. By this I don't mean
  necessarily a service, the Jazz exploration is currently served from GitHub
  Pages. Rather this would be the shell that can load the compiled content and
  render it. Styling, keeping track of what was read, etc. would be handled
  here.

Of course, the current implementation is coupled with the Jazz theme.
Generalizing will take some effort.

Orthogonal to this, the claim checker is also interesting and might be useful.
That can be applied to any LLM (or human) generated document.

All in all, this has been a very fun weekend project.
