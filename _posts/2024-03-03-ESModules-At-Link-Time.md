---
title: Rewriting ESModules at link time in Scala JS
---

## TL:DR

https://quafadas.github.io/ShoelaceSansBundler/

https://github.com/Quafadas/ShoelaceSansBundler

- One scala file
- Scala-cli only
- No bundler
- with shoelace webcomponents...

## Why?
The key idea is to take advantage of the fact that:

1. browsers are already able to load JavaScript ES modules from a URL
2. JavaScript CDNs such as unpkg and jsDelivr already serve NPM packages as ES modules from URLs

## Backstory

A couple of years back, Arman and I had a conversation about scala JS imports, out of which came the odd notion, that the rather odd looking `@JSImport("http://cdn/esModule/import/my/library", "default")` turned out to import an ES Module. As a hardcoded ES module, this has serious shortcomings - notably that obtaining the ESModules was tied to the _library_ artifact.

Arman wrote an SBT plugin which interfaced with the linker circumventing this shortcoming. This allowed mappings to be specified at application link time in SBT... dealt with the knarly JSLinker interface... and ["proved the concept"](https://github.com/armanbilge/scalajs-importmap)

Thanks :-)...

However, I believe the ideal application would be [to have it in scala-cli](https://github.com/VirtusLab/scala-cli/discussions/1968#discussioncomment-5446977)

## Status:
[Merged!](https://github.com/VirtusLab/scala-cli/pull/2737)

Some adjustments to [scala-js-cli](https://github.com/VirtusLab/scala-js-cli/pull/47) were needed, as well as some changes to scala-cli itself.

## Usage Notes
Armans idea ended up being to represent this accordsing to the import map, [supported by browsers](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script/type/importmap#import_map_json_representation).

An example `importmap.json` file, which we would reference in scala-cli, could look like this. The example below which would resolve shoelace components from a CDN, as they are needed.

```json
{
  "imports": {
    "@shoelace-style/shoelace/dist/": "https://cdn.jsdelivr.net/npm/@shoelace-style/shoelace@2.13.1/cdn/"
  }
}
```
This example [shoelace playtime](https://github.com/Quafadas/ShoelaceSansBundler) project, demonstrates that it works. It references the a (prototype) shoelace component facade, and loads the Shoelace `Input` component as an ESModule. The scalacode that references a button component in the UI library, looks something like.

```scala
  @JSImport("@shoelace-style/shoelace/dist/components/button/button.js", JSImport.Namespace)
  @js.native object RawImport extends js.Object
```
The PR, and associated config remaps the import at link time to to be;
`"https://cdn.jsdelivr.net/npm/@shoelace-style/shoelace@2.13.1/cdn/components/button/button.js"`

The nice part, is that once the remapping is in place... at the application use site it's a rather convienient to simply using the component as if it were a native scala.js component.

```scala
  val button = Button()
  button.addEventListener("click", _ => println("Hello, world!"))
  document.body.appendChild(button)
```

Note: Scala JS facade authors could choose (for bonus points) to publish the required import map in their library documentation.

## Use Cases

Currently, I see three attractive use cases for this.

1. Facade construction
2. Testing
3. "on ramp"

All of which are targeted at keeping people  "in the small" for longer, before they need to wheel in the existing, excellent-but-complex toolchain.

### Use cases: On ramp

This is the attractive one for me. Scala-js toolchain is relatively involved.

<pre class="mermaid">
flowchart LR
  compile --> link --> bundle --> serve --> reload
</pre>

When I started, it took some time, to get used to this, what is connceted to what, etc.. When things go wrong, do I need to look at sbt, vite or what? My understanding ended up something like;

<pre class="mermaid">
flowchart LR
sbt --> vite
vite --> sbt
vite --> browser
browser --> vite

</pre>

As far as I can tell, vite is watching files for when they change. When they change it contacts sbt / mill. Now, I think we can simplify that to;

<pre class="mermaid">
flowchart LR
  scala-cli --> live-server
  live-server --> browser
  browser --> live-server
</pre>

Where the live server is a vscode extension / intellij static site which reloads on change.

I'm not an eco-system expert - I could have missed something and feedback is welcome - but before mentally flaming this - please read "non-goals".

### Use cases: Facade construction

This sweeps SBT / vite out the way, and the speed at which one can get started with scala-cli, makes, IMHO, facade construction more attractive and easier to swallow in "side projects" to the main application.

Said differently, this allows one to "experiment in the small".

### Use cases: Testing

Because there's no bi-directional communication between vite / sbt, we are reloading linked artefacts into the browser as a static site, with ESModules resolving in browser.

There's no bundling step involved, which mean no process syncronisation, so I think UI becomes "unit testable" via playwrite, rather than "integration testable".

I'm at the beginning of exploring this, but if it works as well as I believe, it will fundamentally change the way I test frontend code.

## Non-goals

It is categorically _not_ a goal to replace the existing vite / sbt infrastructure. This is intended to _compliment_ to it. Any work done via this mechanism, can be trivially re-used, either through a bundler or the SBT plugin.

It is hoped, that a new user of scala-js, may be able to experience the joys of scala on the frontend, faster and more simply during an experimentation phase, before wheeling in the existing, excellent-but-complex toolchain.

## TODO
Mill plugin


<script src="https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.min.js"></script>
<script>
  mermaid.initialize({ startOnLoad: true });
</script>
