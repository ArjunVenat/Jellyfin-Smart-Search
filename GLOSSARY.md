# Jellyfin Smart Search

Finding moments inside the videos of a personal Jellyfin library by describing them in plain words.

## Language

### What search returns

**Moment**:
The thing a user is looking for and gets back from a search: a point inside a **Media item**, identified by that item and a timestamp.
_Avoid_: Scene (as a name for a result), clip, hit

**Media item**:
A single movie or TV episode in the user's Jellyfin library.
_Avoid_: Video, file, title (a title is metadata, not the item)

### What gets indexed

**Segment**:
The unit of a **Media item** that is indexed and ranked; a **Moment** points into one. Whether a segment is a **Shot**, a **Scene**, or a fixed window is an open decision.
_Avoid_: Chunk, clip

**Shot**:
In the film sense: one continuous camera take, bounded by two cuts.
_Avoid_: Using it to mean the indexed unit (that is a **Segment**)

**Scene**:
In the film sense: a group of **Shots** in one place and continuous time.
_Avoid_: Using it to mean the indexed unit or a search result
