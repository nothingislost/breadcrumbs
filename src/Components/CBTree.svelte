<script lang="ts">
  import { info } from "loglevel";
  import type { MarkdownPostProcessorContext } from "obsidian";
  import { isInVault, openOrSwitch } from "obsidian-community-lib/dist/utils";
  import {
    dfsAllPaths,
    getOppDir,
    getReflexiveClosure,
    getSubInDirs,
  } from "../graphUtils";
  import type { Directions } from "../interfaces";
  import type BCPlugin from "../main";
  import { dropDendron } from "../sharedFunctions";
  import RenderMarkdown from "./RenderMarkdown.svelte";

  export let plugin: BCPlugin;
  export let ctx: MarkdownPostProcessorContext;
  export let el: HTMLElement;
  export let dir: Directions;
  export let fields: string[];
  export let title: string;
  export let depth: string[];
  export let flat: string;
  export let content: string;

  const { settings, app, mainG } = plugin;
  const { sourcePath } = ctx;
  const currFile = app.metadataCache.getFirstLinkpathDest(sourcePath, "");
  const { userHiers } = settings;
  const { basename } = currFile;

  let min = 0,
    max = Infinity;

  if (depth !== undefined) {
    const minNum = parseInt(depth[0]);
    if (!isNaN(minNum)) min = minNum;
    const maxNum = parseInt(depth[1]);
    if (!isNaN(maxNum)) max = maxNum;
  }

  const oppDir = getOppDir(dir);
  const upnDown = getSubInDirs(mainG, dir, oppDir);
  const closed = getReflexiveClosure(upnDown, userHiers);
  const down = getSubInDirs(closed, dir);

  const allPaths = dfsAllPaths(down, basename);
  const index = plugin.createIndex(allPaths, false);
  info({ allPaths, index });

  const lines = index
    .split("\n")
    .map((line) => {
      const pair = line.split("- ");
      return [flat === "true" ? "" : pair[0], pair.slice(1).join("- ")] as [
        string,
        string
      ];
    })
    .filter((pair) => pair[1] !== "");
</script>

{#if title !== "false"}
  <h3>{dir} of {basename}</h3>
{/if}
<div class="BC-tree">
  {#each lines as [indent, link]}
    {#if indent.length / 2 <= max && indent.length / 2 >= min}
      {#if content === "open" || content === "closed"}
        <div>
          <pre class="indent">{indent}</pre>
          <details open={content === "open"}>
            <summary>
              <span
                class="internal-link"
                on:click={async (e) => await openOrSwitch(plugin.app, link, e)}
                on:mouseover={(e) => {
                  //   hoverPreview needs an itemView so it can access `app`...
                  //   hoverPreview(e, el, link)
                }}
              >
                <!-- svelte-ignore a11y-missing-attribute -->
                <a
                  class="internal-link {isInVault(plugin.app, link)
                    ? ''
                    : 'is-unresolved'}">{dropDendron(link, settings)}</a
                >
              </span>
            </summary>
            <RenderMarkdown {app} path={link} />
          </details>
        </div>
      {:else}
        <div>
          <pre class="indent">{indent + "-"}</pre>
          <span
            class="internal-link"
            on:click={async (e) => await openOrSwitch(plugin.app, link, e)}
            on:mouseover={(e) => {
              //   hoverPreview needs an itemView so it can access `app`...
              //   hoverPreview(e, el, link)
            }}
          >
            <!-- svelte-ignore a11y-missing-attribute -->
            <a
              class="internal-link {isInVault(plugin.app, link)
                ? ''
                : 'is-unresolved'}">{dropDendron(link, settings)}</a
            >
          </span>
        </div>
      {/if}
    {/if}
  {/each}
</div>

<style>
  .BC-tree {
    padding-left: 5px;
  }
  /* .BC-tree > div {
    white-space: nowrap;
  } */
  pre.indent {
    display: inline;
    background-color: transparent;
    position: top;
  }
  details {
    display: inline-block;
  }

  .is-unresolved {
    color: var(--text-muted);
  }

  button.append-content {
    padding: 1px 5px;
    margin-right: 2px;
  }
</style>
