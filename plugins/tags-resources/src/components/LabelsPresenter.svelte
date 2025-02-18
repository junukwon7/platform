<script lang="ts">
  import { Doc, WithLookup } from '@hcengineering/core'
  import { createQuery } from '@hcengineering/presentation'
  import type { TagReference } from '@hcengineering/tags'
  import tags from '@hcengineering/tags'
  import { getEventPopupPositionElement, resizeObserver, showPopup, tooltip } from '@hcengineering/ui'
  import { afterUpdate, createEventDispatcher } from 'svelte'
  import TagReferencePresenter from './TagReferencePresenter.svelte'
  import TagsReferencePresenter from './TagsReferencePresenter.svelte'
  import TagsEditorPopup from './TagsEditorPopup.svelte'
  import TagsItemPresenter from './TagsItemPresenter.svelte'
  import LabelsPresenter from './LabelsPresenter.svelte'

  export let value: number
  export let object: WithLookup<Doc>
  export let full: boolean
  export let ckeckFilled: boolean = false
  export let kind: 'short' | 'full' | 'list' | 'link' | 'todo' | 'todo-compact' = 'short'
  export let isEditable: boolean = false
  export let action: (evt: MouseEvent) => Promise<void> | void = async () => {}
  export let compression: boolean = false
  export let ignoreFirst: boolean = false

  const dispatch = createEventDispatcher()

  let items: TagReference[] = []
  const query = createQuery()

  $: update(object, value)

  function update (object: WithLookup<Doc>, value: number) {
    if (value > 0) {
      query.query(tags.class.TagReference, { attachedTo: object._id }, (result) => {
        items = result.slice(ignoreFirst ? 1 : 0)
      })
    } else {
      query.unsubscribe()
      items = []
    }
  }

  async function tagsHandler (evt: MouseEvent): Promise<void> {
    showPopup(TagsEditorPopup, { object }, getEventPopupPositionElement(evt))
  }

  let allWidth: number
  const widths: number[] = []

  afterUpdate(() => {
    let count: number = 0
    widths.forEach((i) => (count += i))
    full = count > allWidth
    dispatch('change', { full, ckeckFilled })
  })
</script>

{#if kind === 'list' || kind === 'link'}
  {#if items.length > 4}
    <div
      class="label-box no-shrink"
      use:tooltip={{
        component: TagsItemPresenter,
        props: { value: items, kind: 'link' }
      }}
    >
      <TagsReferencePresenter {items} {kind} />
    </div>
  {:else}
    {#each items as value}
      <div class="label-box no-shrink" title={value.title}>
        <TagReferencePresenter attr={undefined} {value} {kind} />
      </div>
    {/each}
  {/if}
{:else if kind === 'todo'}
  <div class="flex-row-top flex-wrap flex-gap-0-5">
    {#each items as value}
      <TagReferencePresenter attr={undefined} {value} {kind} />
    {/each}
  </div>
{:else if kind === 'todo-compact'}
  {#if items.length > 1}
    <div class="flex-row-top flex-wrap flex-gap-0-5">
      <TagReferencePresenter attr={undefined} value={items[0]} kind={'todo'} />
      <div
        class="todoLabel-plus-container font-medium-11"
        use:tooltip={{
          component: LabelsPresenter,
          props: { value, object, isEditable, kind: 'todo', ignoreFirst: true }
        }}
      >
        +{items.length - 1}
      </div>
    </div>
  {:else if items.length === 1}
    <TagReferencePresenter attr={undefined} value={items[0]} kind={'todo'} />
  {/if}
{:else}
  <!-- svelte-ignore a11y-click-events-have-key-events -->
  <!-- svelte-ignore a11y-no-static-element-interactions -->
  <div
    class="labels-container"
    style:justify-content={kind === 'short' ? 'space-between' : 'flex-start'}
    class:w-full={kind === 'full'}
    style:flex-wrap={kind === 'short' || compression ? 'nowrap' : 'wrap'}
    style:flex-shrink={compression ? 1 : 0}
    use:resizeObserver={(element) => {
      allWidth = element.clientWidth
    }}
    on:click|stopPropagation={(evt) => {
      if (isEditable) tagsHandler(evt)
      else action(evt)
    }}
  >
    {#each items as value, i}
      <div class="label-box wrap-{kind}" title={value.title}>
        <TagReferencePresenter attr={undefined} {value} kind={'link'} bind:realWidth={widths[i]} />
      </div>
    {/each}
  </div>
{/if}

<style lang="scss">
  .todoLabel-plus-container {
    padding: var(--spacing-0_25) var(--spacing-0_75);
    color: var(--tag-on-subtle-PorpoiseText);
    background-color: var(--tag-subtle-PorpoiseBackground);
    border-radius: var(--extra-small-BorderRadius);
  }
  .labels-container {
    overflow: hidden;
    display: flex;
    align-items: center;
    flex-shrink: 1;
    min-width: 0;
    border-radius: 0.25rem;
  }

  .label-box {
    display: flex;
    align-items: center;
    width: auto;
    min-width: 0;
    border-radius: 0.25rem;
    transition: box-shadow 0.15s ease-in-out;
  }
  .wrap-short:not(:last-child) {
    margin-right: 0.375rem;
  }
  .wrap-full {
    margin: 0.125rem;
  }
</style>
