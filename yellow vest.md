# Who wears the yellow vest in your AI?

A while back I saw [a prank](https://youtu.be/GyvRamX1VyA). Some people put on a hi-vis yellow vest and walked into movie theaters, restaurants, museums. Nobody stopped them. They looked like maintenance, and nobody questions maintenance. They walked in wherever they wanted, like in a spy movie.

That stuck with me. In a software system, what is it that gives us the feeling of being secure? A username and password? A JWT token? Getting into the server over SSH with a key only? They are all solid answers, and most of them fall the same way. Somebody drops a USB stick wearing a yellow vest in the company parking lot, an employee plugs it in, and they are inside.

With LLMs we do not even have that. There is no password to check, no key to rotate. We send everything to a provider and trust that it behaves.

And what if the data is sensitive? Name, phone, email. You would have to process it, obfuscate it before sending. Now think about a project in healthcare, or in finance, where taking care of that data starts to matter.

## What near.ai does

I know [near.ai](https://near.ai) is an LLM gateway. It runs prompts anonymously, encrypted, and with attestation you can verify it actually did that.

It feels like a VPN for LLMs to me. I know technically it is a different thing, but they share the idea. From the outside you cannot tell what is being processed or with what data.

Let us get a bit technical, because there are two kinds of models. Open models run inside a TEE, the prompt is decrypted inside the enclave, so nobody sees it. The enclave is a slice of memory the CPU isolates from the rest of the machine, neither the operating system nor whoever administers the server can look at what is in there. Private models are anonymous, they use a shared key so the provider does not know your identity, but it does receive your prompt and processes it.
If this were mail, the TEE is a sealed letter that arrives unopened. The anonymous one is an open letter with no return address.

Then there is attestation, which sounds like a weird word. It is there so that, with a signed cryptographic proof, you can verify the execution happened inside the enclave the way you expected. That is the trust the other gateways are missing.

So picking a model is not a config detail. With an anonymous model the provider does not know who you are, but it reads everything you send, and if your messages follow a pattern it can reconstruct you anyway. The TEE is the only one of the two where the prompt is never visible. If the data is sensitive, the letter goes sealed or it does not go.

That said, the yellow vest can still sneak in some other way. The harnesses, the agents, how you write the prompt. Any of those leaks secrets without meaning to.

## Pushing the limits

While I was at it, I wanted to see how far the agents running on the gateway would hold up. I tried GPT-5-mini (anonymous) and google/gemma-4-31B-it (TEE), both as text and image models.
First I am going to generate the key at [cloud.near.ai](https://cloud.near.ai) and then use it in three different agents: Hermes, Pi Agent and OpenClaw.
For that go into keys and hit new key, give it a name and a limit, which is optional, and then the key is shown only once, careful, copy it.

![near ai](image.png)

### Hermes

Starting with [Hermes](https://github.com/NousResearch/hermes-agent) was a mistake, it works so well that I just installed it and that was it. After installing it I ran `hermes model`, picked "your own endpoint" and pasted the URL, which is https://cloud-api.near.ai/v1, and the key, and that was that. The rest of the tools are very intuitive and there is nothing else to touch.

From the dashboard I asked it for simple things. Search for Pokémon cards on Amazon and put together a list. A chocolate cake recipe. I asked it what was in a photo. All fine. Since I had to head out, I hooked it up to Telegram to keep going from my phone.
That is where I started really hunting for the limit. I sent it an audio message asking it to search for Pokémon cards again, expecting an "I cannot listen to audio". It just did it, no complaints. Then I asked it to answer me out loud and it wrote itself a Python script to do that. Last, I asked for a picture of a puppy, and apparently it called a five year old over to draw a dog in Paint.

![Three dogs](./puppies.png)

The day I get to build my own R2-D2, this is one of the models I would use.

### Pi Agent

Then I went through [Pi Agent](https://pi.dev/). This one is a bit more complex to set up, you have to edit `~/.pi/agent/models.json` and put in something like this:

```json
{
  "providers": {
    "near-ai": {
      "name": "NEAR AI Cloud",
      "baseUrl": "https://cloud-api.near.ai/v1",
      "apiKey": "$NEAR_AI_API_KEY",
      "api": "openai-completions",
      "models": [
        {
          "id": "openai/gpt-5-mini",
          "name": "GPT-5 Mini (near.ai)",
          "reasoning": true,
          "input": ["text"],
          "cost": { "input": 0.25, "output": 2.0 },
          "contextWindow": 400000,
          "maxTokens": 16384
        }
      ]
    }
  }
}
```

My understanding is that it is the base OpenClaw was built on, and this time I used it for code. It struck me as simple and minimal. It is like going from Eclipse or JetBrains to VS Code. You open it, it is fast, it has nothing bolted on, and you head into plugins to install everything, down to the Pokémon VS Code one that is useless but is right there.

I used it for code and it works really well. The feeling I am left with is that it competes with [opencode](https://opencode.ai), or that it is the base for a harness.

### OpenClaw

Last I tried [OpenClaw](https://github.com/openclaw/openclaw) and it let me down. To start with, to configure it I ran `openclaw config`, picked local as the gateway, then went into model, picked "more", then custom provider and pasted the URL https://cloud-api.near.ai/ and the key. Then you pick OpenAI-compatible and type a model like openai/gpt-5-mini, verify and done.
I never got it configured right, the model hallucinated and I never managed to send it audio. I repeated everything I had done with Hermes and the experience did not come close. I gave it a screenshot of the spending dashboard and asked what it saw. It started describing all sorts of things and ended up spitting out the letter "a" for twenty lines straight.

![OpenClaw response](./openclaw.png)

## Back to the vest

I ran the attestation. The signature validates and the measurements check out.
Verifying at the hardware level means you stop running into maintenance people in yellow vests. You no longer believe their lies, you move on to verifying they are telling the truth.
On performance and cost it comes out even. It should take a bit longer, but with agents you do not feel it, it is like using ChatGPT, Claude or any of them.
I would use it on projects where privacy and data protection are the main thing. Or to try out different models, since for the same price I can use models from Claude, OpenAI, Google, DeepSeek, Kimi and GLM.
Next time I will talk about the ones carrying ladders.

## Appendix, how attestation works

1. **The hardware measures what runs.** The CPU (Intel TDX) computes hashes of the firmware, kernel and image loaded inside the enclave. Those hashes go into registers only the CPU can write (MRTD, RTMR0 through RTMR3). Nobody on the inside can fake them.
2. **The CPU signs those measurements.** It generates a quote with the measurements plus 64 bytes of `report_data` that you choose, all signed with a key that ships from the factory and chains up to an Intel CA.
3. **You verify the signature against the manufacturer root.** If it validates, you know the quote was issued by real silicon. For that you need collateral (CRLs, TCB info) fetched from the PCCS.
4. **You compare the measurements against what you expected.** The signature alone is not enough, it proves something genuine ran, not that your code ran. The value comes from comparing RTMR and MRTD against the reproducible build.
