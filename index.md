# AutoUI

*The DOMancer: summon a user interface from an empty page*

This demo runs an [OpenRouter](https://openrouter.ai/) model in a tiny
JavaScript loop on a blank HTML page, giving it only the ability to
`eval()` expressions. The model figures out the setup, builds a DOM
interface from scratch to talk to you, and can create other impressive
graphical HTML toys on the fly.

Start by checking out the simple [source code](go.txt), then enter a
valid API key, choose a model, open the browser console to keep an eye
on the model's activity, and hit Go. Leave the key blank to provision
one using OpenRouter's [OAuth
PKCE](https://openrouter.ai/docs/guides/overview/auth/oauth).

<style>
#launcher {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5em;
  input, select, button {
    appearance: none;
    background: none;
    border: 2px solid #9bf;
    border-radius: 0.5em;
    box-sizing: border-box;
    font-family: ui-monospace, monospace;
    font-size: inherit;
    max-width: 100%;
    padding: 0.5em;
    &:user-invalid {
      border: 2px solid #f88;
    }
  }
}
</style>

<form id="launcher">
  <input id="key" placeholder="API key" size="80" type="text">
  <input id="model" list="models" placeholder="Model" size="24" type="text">
  <datalist id="models"></datalist>
  <select id="reasoning">
    <option value="">Default</option>
    <option value="none">None</option>
    <option value="minimal">Minimal</option>
    <option value="low">Low</option>
    <option value="medium">Medium</option>
    <option value="high">High</option>
    <option value="xhigh">Extra</option>
    <option value="max">Maximum</option>
  </select>
  <select id="search">
    <option value="">Search off</option>
    <option value="1">Search on</option>
  </select>
  <button id="go" type="button">Go</button>
</form>

<script type="module">
const [go, key, model, models, reasoning, search]
  = ['go', 'key', 'model', 'models', 'reasoning', 'search']
    .map(id => document.getElementById(id));
const code = new URLSearchParams(location.search).get('code');
const verifier = sessionStorage.getItem('verifier');

if (code && verifier) {
  history.replaceState(null, '', location.origin + location.pathname);
  sessionStorage.removeItem('verifier');

  const response = await fetch('https://openrouter.ai/api/v1/auth/keys', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      code: code,
      code_challenge_method: 'S256',
      code_verifier: verifier
    })
  });

  if (response.ok)
    localStorage.setItem('key', (await response.json()).key);
}

key.value = localStorage.getItem('key') ?? '';
model.value = localStorage.getItem('model') ?? '';
reasoning.value = localStorage.getItem('reasoning') ?? '';
search.value = localStorage.getItem('search') ?? '';
[key, model].forEach(node => node.setCustomValidity(''));

try {
  const response = await fetch('https://openrouter.ai/api/v1/models');
  if (response.ok) {
    const { data } = await response.json();
    models.replaceChildren(...data.map(({ id }) => id).sort()
      .filter(id => !id.endsWith(':batch')).map(id => new Option(id)));
  }
} catch {}

key.addEventListener('change', async () => {
  if (key.value.startsWith('sk-or-')) {
    try {
      const response = await fetch('https://openrouter.ai/api/v1/auth/key', {
        headers: { Authorization: `Bearer ${key.value}`, }
      });
      if (response.ok) {
        localStorage.setItem('key', key.value);
        return key.setCustomValidity('');
      }
    } catch {}
  }
  return key.setCustomValidity(key.value ? 'Invalid API key' : '');
});

model.addEventListener('change', () => {
  if ([...models.options].some(option => option.value == model.value)) {
    localStorage.setItem('model', model.value);
    return model.setCustomValidity('');
  }
  return model.setCustomValidity('Invalid model ID');
});

reasoning.addEventListener('change', () => {
  localStorage.setItem('reasoning', reasoning.value);
});

search.addEventListener('change', () => {
  localStorage.setItem('search', search.value);
});

go.addEventListener('click', async () => {
  if (key.value) {
    [key, model].forEach(node => node.dispatchEvent(new Event('change')));
    if (key.checkValidity() && model.checkValidity())
      location.assign(`go.html#key=${encodeURIComponent(key.value)}`
        + `&model=${encodeURIComponent(model.value)}`
        + (reasoning.value ? `&reasoning=${reasoning.value}` : '')
        + (search.value ? '&search' : ''));
  } else {
    const verifier = crypto.getRandomValues(new Uint8Array(32)).toBase64(
      { alphabet: 'base64url', omitPadding: true });
    const challenge = new Uint8Array(await crypto.subtle.digest('SHA-256',
        new TextEncoder().encode(verifier))).toBase64(
      { alphabet: 'base64url', omitPadding: true });
    sessionStorage.setItem('verifier', verifier);

    const url = new URL('https://openrouter.ai/auth');
    url.searchParams.set('callback_url', location.origin + location.pathname);
    url.searchParams.set('code_challenge', challenge);
    url.searchParams.set('code_challenge_method', 'S256');
    url.searchParams.set('key_label', 'AutoUI');
    location.assign(url);
  }
});
</script>

## Notes

Recent but cheap models like *openai/gpt-6-luna*, *openai/gpt-6-luna-pro*,
and *anthropic/claude-haiku-5.5* are a good starting point; it typically
costs around $0.0005 to one-shot an inviting chat interface with Luna 6,
and around $0.0007 with Haiku 5.5. Default reasoning is usually fine.

To open the console on a desktop browser, use the following keyboard
shortcuts:

| Browser  | Linux           | macOS           | Windows         |
|:---------|:---------------:|:---------------:|:---------------:|
| Chromium | &#8963;&#8679;J | &#8997;&#8984;J | &#8963;&#8679;J |
| Firefox  | &#8963;&#8679;K | &#8997;&#8984;K | &#8963;&#8679;K |
| Safari   | &mdash;         | &#8997;&#8984;C | &mdash;         |

With Safari, you might need to enable *Show features for web developers*
first. Other browsers are less patronising. The demo works fine on
mobile devices too, but you can't easily watch the console to monitor
progress there.

The blank page sits apparently idle for a short time while the model
reasons about the setup, and larger models can think for thirty seconds
or more before returning with something imaginative. If the API call
drops or fails, or if any other error occurs, a dialog box reports this.
Reloading the page restarts generation from scratch, so if you get a
boring first turn, it's worth retrying.

Most modern models quickly understand the scenario, and correctly block
awaiting user input where appropriate. There is a small risk of a confused
model calling the API in a busy loop, so it's best to limit the amount of
credit available to the key you use, and keep an eye on activity in the
console at first. After a model has successfully waited, it's unlikely
to end up spinning on a later turn.

Once you can communicate, you can say things like "Let's play Doom!"
or "How about a 3D simulation of a galaxy?" or "Can you build a musical
instrument?" or "Please surprise me!" with invariably impressive results.

At the time of writing, *anthropic/claude-opus-5.5* is thwarted by
Anthropic's censorship cult, right after reading the page's source code.
Presumably this is a false positive, but who knows how far these hysterical
people have stretched their fear of dangerous thoughts?

You don't need to use the client-side form above to run the demo. It's a
simple static page and just expects `key` and `model` as URI-encoded
parameters in the hash part of the URL. Usually neither the key nor the
model slug requires escaping. You can also specify `api` to override
the responses endpoint, `reasoning` to set the reasoning effort level,
and `search` to enable the native web search tool. The page runs fine
downloaded to a local file.

An early prototype exposed `eval()` as a custom tool, but this encouraged
smaller models to address the user with assistant replies even when
directly told or shown that the harness ignores them. Repurposing the
main communication channel for JavaScript worked much better.

Implementing PKCE for the form above turned out to be surprisingly simple.
Inspect the source of this page or a [stand-alone version](pkce.txt)
for the details. The code is in two halves: the request happens in the Go
button click handler if `key.value` is empty, and the callback is dealt
with at top level when the module is evaluated.

If you enjoyed this demo, you might also be interested in
[ua](https://github.com/arachsys/ua).

## Contact

This toy was hand-made by [Chris Webb](https://arachsys.github.io/)
\<[chris@arachsys.com](mailto:chris@arachsys.com)> and is distributed as
Free Software under the terms of the [MIT licence](copying.txt). Have fun!
