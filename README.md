<p align="center" style="font-weight: bold">[Part 1](https://github.com/bespoken/super-simple-audio-player/blob/master/README.md) | Part 2 | Part 3 - Coming Soon!</p>

# Part 2 - Next/Previous and the AudioPlayer Queue
In this step, we show you how to use the builtin AMAZON.NextIntent and AMAZON.PreviousIntent.

## Managing the Queue
To manage the queue, we are going to make make use of the playBehavior property. This is what determines whether a track is played immediately or is queued to be played once the current audio completes.

Rather than always saying `REPLACE_ALL` for the playBehavior, as we did in Part 1, we will also make use of `ENQUEUE`:

To do this, we expose a few more parameters on our `play` function:
```
SimplePlayer.prototype.play = function (audioURL, offsetInMilliseconds, playBehavior, token, previousToken) {
    var response = {
        version: "1.0",
        response: {
            shouldEndSession: true,
            directives: [
                {
                    type: "AudioPlayer.Play",
                    playBehavior: playBehavior,
                    audioItem: {
                        stream: {
                            url: audioURL,
                            token: token, // Unique token for the track - needed when queueing multiple tracks
                            expectedPreviousToken: previousToken, // The expected previous token - when using queues, ensures safety
                            offsetInMilliseconds: offsetInMilliseconds
                        }
                    }
                }
            ]
        }
    };

    this.context.succeed(response);
};
```
We added three parameters - `playBehavior`, `token`, and `previousToken`.

The `playBehavior` can be REPLACE_ALL, ENQUEUE, or REPLACE_ENQUEUED.
* REPLACE_ALL causes the track to begin playing immediately
* ENQUEUE causes the track to be added at the end of the queue
* REPLACE_ENQUEUED causes the track to be played next (and the rest of the queue to cleared)

For our purposes, we are only using REPLACE_ALL and ENQUEUE right now.

The tokens are also important. They give a unique label to the track being played.
They can be any unique value, but here we use a simple numbering scheme based on our array of podcasts.
The expectedPreviousToken is set to prevent race conditions, where multiple tracks may be getting enqueued simultaneously.
You can [read more about it here](https://developer.amazon.com/public/solutions/alexa/alexa-skills-kit/docs/custom-audioplayer-interface-reference#play) - the key thing to keep in mind with is is that:
 * It must be set properly for calls to ENQUEUE
 * It does not need to be set for calls with REPLACE_ALL or REPLACE_ENQUEUED

## Handling the Intents
We use the builtin AMAZON.NextIntent and AMAZON.PreviousIntent.

We now have a an array of podcasts to playback, and we move through them in a cycle. Meaning, when reach the end of the array, we start over at the beginning.
```
// If we have reached the end of the feed, start back at the beginning
podcastIndex >= podcastFeed.length - 1 ? podcastIndex = 0 : podcastIndex++;

this.queue(podcastFeed[podcastIndex], 0, "REPLACE_ALL", podcastIndex);
```
 That way, a user can keep saying "Next" or "Previous" perpetually.

## Handling the AudioPlayer.PlaybackNearlyFinished
This is an important one - besides allowing the user to say Next and Previous, we want to also automatically start our next podcast once the current one finishes.

To do this, we take advantage of the AudioPlayer.PlaybackNearlyFinished request.
Since only one track can be queued at a time, we need to use this to create our queue (otherwise, we might just load up our queue with a single directive at the start).

This request comes near the conclusion of the current audio playing. Here we:
* Set the playBehavior as `ENQUEUE`
* Set the previousToken to the currently playing track

By setting all this properly, we ensure our audio smoothly transitions from one track to the next. Neat, right?

## What Is Next?
Our AudioPlayer skill is getting more complicated. To make sure it's all working correctly without lots of tedious manual testing,
in our next edition we will add unit tests with [the BSTAlexa emulator](http://docs.bespoken.tools/en/latest/tutorials/tutorial_bst_emulator_nodejs/).
