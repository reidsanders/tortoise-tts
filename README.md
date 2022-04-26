# TorToiSe

Tortoise is a text-to-speech program built with the following priorities:

1. Strong multi-voice capabilities.
2. Highly realistic prosody and intonation.

This repo contains all the code needed to run Tortoise TTS in inference mode.

## What's in a name?

I'm naming my speech-related repos after Mojave desert flora and fauna. Tortoise is a bit tongue in cheek: this model
is insanely slow. It leverages both an autoregressive decoder **and** a diffusion decoder; both known for their low
sampling rates. On a K80, expect to generate a medium sized sentence every 2 minutes.

## Demos

See [this page](http://nonint.com/static/tortoise_v2_examples.html) for a large list of example outputs.

## Usage guide

### Colab

Colab is the easiest way to try this out. I've put together a notebook you can use here:
https://colab.research.google.com/drive/1wVVqUPqwiDBUVeWWOUNglpGhU3hg_cbR?usp=sharing

### Installation

If you want to use this on your own computer, you must have an NVIDIA GPU. Installation:

```shell
git clone https://github.com/neonbjb/tortoise-tts.git
cd tortoise-tts
pip install -r requirements.txt
```

### do_tts.py

This script allows you to speak a single phrase with one or more voices.
```shell
python do_tts.py --text "I'm going to speak this" --voice dotrice --preset fast
```

### read.py

This script provides tools for reading large amounts of text.

```shell
python read.py --textfile <your text to be read> --voice dotrice
```

This will break up the textfile into sentences, and then convert them to speech one at a time. It will output a series 
of spoken clips as they are generated. Once all the clips are generated, it will combine them into a single file and
output that as well.

Sometimes Tortoise screws up an output. You can re-generate any bad clips by re-running `read.py` with the --regenerate
argument.

### API

Tortoise can be used programmatically, like so:

```python
reference_clips = [utils.audio.load_audio(p, 22050) for p in clips_paths]
tts = api.TextToSpeech()
pcm_audio = tts.tts_with_preset("your text here", reference_clips, preset='fast')
```

## Voice customization guide

Tortoise was specifically trained to be a multi-speaker model. It accomplishes this by consulting reference clips.

These reference clips are recordings of a speaker that you provide to guide speech generation. These clips are used to determine many properties of the output, such as the pitch and tone of the voice, speaking speed, and even speaking defects like a lisp or stuttering. The reference clip is also used to determine non-voice related aspects of the audio output like volume, background noise, recording quality and reverb.

### Provided voices

This repo comes with several pre-packaged voices. You will be familiar with many of them. :)

Most of the provided voices were not found in the training set. Experimentally, it seems that voices from the training set
produce more realistic outputs then those outside of the training set. The following voices come from the training set:
 atkins, dotrice, grace, harris, kennard, lescault, mol, otto.

### Adding a new voice

To add new voices to Tortoise, you will need to do the following:

1. Gather audio clips of your speaker(s). Good sources are YouTube interviews (you can use youtube-dl to fetch the audio), audiobooks or podcasts. Guidelines for good clips are in the next section.
2. Cut your clips into ~10 second segments. You want at least 3 clips. More is better, but I only experimented with up to 5 in my testing.
3. Save the clips as a WAV file with floating point format and a 22,050 sample rate.
4. Create a subdirectory in voices/
5. Put your clips in that subdirectory.
6. Run tortoise utilities with --voice=<your_subdirectory_name>.

### Picking good reference clips

As mentioned above, your reference clips have a profound impact on the output of Tortoise. Following are some tips for picking
good clips:

1. Avoid clips with background music, noise or reverb. These clips were removed from the training dataset. Tortoise is unlikely to do well with them.
2. Avoid speeches. These generally have distortion caused by the amplification system.
3. Avoid clips from phone calls.
4. Avoid clips that have excessive stuttering, stammering or words like "uh" or "like" in them.
5. Try to find clips that are spoken in such a way as you wish your output to sound like. For example, if you want to hear your target voice read an audiobook, try to find clips of them reading a book.
6. The text being spoken in the clips does not matter, but diverse text does seem to perform better.

## Advanced Usage

### Generation settings

Tortoise is primarily an autoregressive decoder model combined with a diffusion model. Both of these have a lot of knobs
that can be turned that I've abstracted away for the sake of ease of use. I did this by generating thousands of clips using
various permutations of the settings and using a metric for voice realism and intelligibility to measure their effects. I've
set the defaults to the best overall settings I was able to find. For specific use-cases, it might be effective to play with
these settings (and it's very likely that I missed something!)

These settings are not available in the normal scripts packaged with Tortoise. They are available, however, in the API. See
```api.tts``` for a full list.

### Playing with the voice latent

Tortoise ingests reference clips by feeding them through individually through a small submodel that produces a point latent, then taking the mean of all of the produced latents. The experimentation I have done has indicated that these point latents are quite expressive, affecting
everything from tone to speaking rate to speech abnormalities.

This lends itself to some neat tricks. For example, you can combine feed two different voices to tortoise and it will output what it thinks the "average" of those two voices sounds like. You could also theoretically build a small extension to Tortoise that gradually shifts the 
latent from one speaker to another, then apply it across a bit of spoken text (something I havent implemented yet, but might
get to soon!) I am sure there are other interesting things that can be done here. Please let me know what you find!

### Send me feedback!

Probabilistic models like Tortoise are best thought of as an "augmented search" - in this case, through the space of possible
utterances of a specific string of text. The impact of community involvement in perusing these spaces (such as is being done with
GPT-3 or CLIP) has really surprised me. If you find something neat that you can do with Tortoise that isn't documented here,
please report it to me! I would be glad to publish it to this page.

## Model architecture

Tortoise TTS is inspired by OpenAI's DALLE, applied to speech data and using a better decoder. It is made up of 5 separate 
models that work together. I've assembled a write-up of the system architecture here:
[https://nonint.com/2022/04/25/tortoise-architectural-design-doc/](https://nonint.com/2022/04/25/tortoise-architectural-design-doc/)

## Training

These models were trained on my "homelab" server with 8 RTX 3090s over the course of several months. They were trained on a dataset consisting of 
~50k hours of speech data, most of which was transcribed by [ocotillo](http://www.github.com/neonbjb/ocotillo). Training was done on my own
[DLAS](https://github.com/neonbjb/DL-Art-School) trainer.

I currently do not have plans to release the training configurations or methodology. See the next section..

## Ethical Considerations

Tortoise v2 works considerably better than I had planned. When I began hearing some of the outputs of the last few versions, I began
wondering whether or not I had an ethically unsound project on my hands. The ways in which a voice-cloning text-to-speech system
could be misused are many. It doesn't take much creativity to think up how.

After some thought, I have decided to go forward with releasing this. Following are the reasons for this choice:

1. It is primarily good at reading books and speaking poetry. Other forms of speech do not work well.
2. It was trained on a dataset which does not have the voices of public figures. While it will attempt to mimic these voices if they are provided as references, it does not do so in such a way that most humans would be fooled.
3. The above points could likely be resolved by scaling up the model and the dataset. For this reason, I am currently withholding details on how I trained the model, pending community feedback.
4. I am releasing a separate classifier model which will tell you whether a given audio clip was generated by Tortoise or not. See `tortoise-detect` above.
5. If I, a tinkerer with a BS in computer science with a ~$15k computer can build this, then any motivated corporation or state can as well. I would prefer that it be in the open and everyone know the kinds of things ML can do.

### Diversity

The diversity expressed by ML models is strongly tied to the datasets they were trained on.

Tortoise was trained primarily on a dataset consisting of audiobooks. I made no effort to
balance diversity in this dataset. For this reason, Tortoise will be particularly poor at generating the voices of minorities
or of people who speak with strong accents.

## Looking forward

Tortoise v2 is about as good as I think I can do in the TTS world with the resources I have access to. A phenomenon that happens when
training very large models is that as parameter count increases, the communication bandwidth needed to support distributed training
of the model increases multiplicatively. On enterprise-grade hardware, this is not an issue: GPUs are attached together with
exceptionally wide buses that can accommodate this bandwidth. I cannot afford enterprise hardware, though, so I am stuck.

I want to mention here
that I think Tortoise could do be a **lot** better. The three major components of Tortoise are either vanilla Transformer Encoder stacks
or Decoder stacks. Both of these types of models have a rich experimental history with scaling in the NLP realm. I see no reason
to believe that the same is not true of TTS.

The largest model in Tortoise v2 is considerably smaller than GPT-2 large. It is 20x smaller that the original DALLE transformer.
Imagine what a TTS model trained at or near GPT-3 or DALLE scale could achieve.

If you are an ethical organization with computational resources to spare interested in seeing what this model could do 
if properly scaled out, please reach out to me! I would love to collaborate on this.

## Notice

Tortoise was built entirely by me using my own hardware. My employer was not involved in any facet of Tortoise's development.

If you use this repo or the ideas therein for your research, please cite it! A bibtex entree can be found in the right pane on GitHub.