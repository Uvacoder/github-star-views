<svelte:options runes />

<script lang="ts">
  import { createCache, createStorageStore } from "minicache";
  import { untrack } from "svelte";
  import { fetchColors, fetchStars, type Stared } from "./lib/fetcher";
  const userInputId = "user";
  let user = $state("");
  let page = $state(1);
  let repos = $state<Promise<Stared[]>>(Promise.resolve([]));
  let colors = $state<Map<string, string>>();
  const cache = createCache(
    createStorageStore(import.meta.env.PROD ? sessionStorage : localStorage)
  );
  const search = (u: string, p: number) => {
    repos = cache.with([u, p], fetchStars).value;
  };

  // on mount
  $effect(() =>
    untrack(() => {
      cache.with(["colors"], fetchColors).value.then((c) => (colors = c));
      const params = new URLSearchParams(window.location.search);
      user = params.get("user") || "";
      page = Number(params.get("page") || 1);
      if (user) search(user, page);
    })
  );
  // sync url
  $effect(() => {
    const params = new URLSearchParams([
      ["user", user],
      ["page", page.toString()],
    ]);
    window.scrollTo(0, 0);
    window.history.replaceState(null, "", `?${params}`);
  });

  const onsubmit = (event: Event) => {
    event.preventDefault();
    page = 1;
    search(user, page);
  };
</script>

<header>
  <h1>Star viewer</h1>
  <form {onsubmit}>
    <label for={userInputId} class="sr-only">Username</label>
    <input
      id={userInputId}
      type="search"
      placeholder="octocat"
      bind:value={user}
    />
    <button type="submit">Search</button>
  </form>
</header>

<main>
  {#await repos}
    <p>Loading...</p>
  {:then data}
    {#each data as repo (repo.id)}
      <section class="repo">
        <h3 class="repo_name">
          <a href={repo.html_url} target="_blank">
            <img
              src="https://github.com/{repo.owner.login}.png?size=16"
              alt=""
              width={16}
              height={16}
              class:rounded={repo.owner.type !== "Organization"}
            />
            {repo.full_name}
          </a>
        </h3>
        <div class="repo_indicator">
          {#if repo.language}
            <small
              class="language"
              style="--language-color:{colors?.get(repo.language) ?? '#333'}"
            >
              {repo.language}
            </small>
          {/if}
          <small>
            <a href="{repo.html_url}/stargers" target="_blank" class="star">
              {repo.stargazers_count.toLocaleString()}
            </a>
          </small>
          {#if repo.license_name}
            <small class="license">{repo.license_name}</small>
          {/if}
        </div>
        <p class="repo_description">{repo.description}</p>
      </section>
    {/each}
  {:catch error}
    <p>{error.message}</p>
  {/await}
  <nav>
    <button
      disabled={page === 1}
      on:click={() => {
        page -= 1;
        search(user, page);
      }}
    >
      &lt;
    </button>
    <button
      on:click={() => {
        page += 1;
        search(user, page);
      }}
    >
      &gt;
    </button>
  </nav>
</main>

<style>
  header {
    display: flex;
    flex-wrap: wrap;
    justify-content: space-between;
  }
  header form {
    display: flex;
    align-self: flex-end;
  }
  .sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    border: 0;
  }
  main {
    display: flex;
    flex-direction: column;
    padding: 1.5em 0;
    gap: 2.5em;
  }
  .repo {
    display: grid;
    grid-template-areas:
      "name indicator"
      "description description";
    grid-template-columns: auto 1fr;
    align-items: baseline;
    row-gap: 1em;
    column-gap: 1.5em;
  }
  @media (max-width: 768px) {
    .repo {
      grid-template-areas:
        "name"
        "indicator"
        "description";
      grid-template-columns: auto 1fr;
    }
  }
  .repo_name {
    grid-area: name;
    margin: 0;
  }
  .repo_indicator {
    grid-area: indicator;
    display: flex;
    column-gap: 1em;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
  .repo_description {
    grid-area: description;
    margin: 0;
  }
  .rounded {
    border-radius: 9999px;
  }
  .language::before {
    content: "";
    display: inline-block;
    width: 0.75em;
    height: 0.75em;
    border-radius: 50%;
    margin-right: 0.5em;
    background-color: var(--language-color);
  }
  .star {
    color: inherit;
  }
  .star::before {
    content: "⭐";
    display: inline-block;
    margin-right: 0.25em;
  }
  .license::before {
    content: "⚖";
    display: inline-block;
    margin-right: 0.25em;
  }
  nav {
    display: flex;
    justify-content: flex-end;
  }
</style>
