<!--
// Copyright © 2020, 2021 Anticrm Platform Contributors.
// Copyright © 2021, 2022 Hardcore Engineering Inc.
//
// Licensed under the Eclipse Public License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License. You may
// obtain a copy of the License at https://www.eclipse.org/legal/epl-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
//
// See the License for the specific language governing permissions and
// limitations under the License.
-->
<script lang="ts">
  import { Class, Doc, Mixin, Ref } from '@hcengineering/core'
  import notification from '@hcengineering/notification'
  import { Panel } from '@hcengineering/panel'
  import { getResource } from '@hcengineering/platform'
  import {
    ActionContext,
    AttributeCategory,
    AttributeCategoryOrder,
    AttributesBar,
    KeyedAttribute,
    createQuery,
    getAttributePresenterClass,
    getClient,
    hasResource
  } from '@hcengineering/presentation'
  import { AnyComponent, Button, Component, IconMixin, IconMoreH } from '@hcengineering/ui'
  import view from '@hcengineering/view'
  import { createEventDispatcher, onDestroy } from 'svelte'

  import { DocNavLink, ParentsNavigator, getDocLabel, getDocMixins, showMenu } from '..'
  import { categorizeFields, getCollectionCounter, getFiltredKeys } from '../utils'
  import DocAttributeBar from './DocAttributeBar.svelte'

  export let _id: Ref<Doc>
  export let _class: Ref<Class<Doc>>
  export let embedded: boolean = false
  export let readonly: boolean = false

  let realObjectClass: Ref<Class<Doc>> = _class
  let lastId: Ref<Doc> = _id
  let object: Doc

  const pClient = getClient()
  const hierarchy = pClient.getHierarchy()
  const inboxClient = getResource(notification.function.GetInboxNotificationsClient).then((res) => res())

  $: read(_id)
  function read (_id: Ref<Doc>): void {
    if (lastId !== _id) {
      const prev = lastId
      lastId = _id
      void inboxClient.then(async (client) => {
        await client.readDoc(pClient, prev)
      })
    }
  }

  onDestroy(async () => {
    await inboxClient.then(async (client) => {
      await client.readDoc(pClient, _id)
    })
  })

  const query = createQuery()
  $: updateQuery(_id, _class)

  function updateQuery (_id: Ref<Doc>, _class: Ref<Class<Doc>>) {
    if (_id && _class) {
      query.query(_class, { _id }, (result) => {
        object = result[0]
        if (object != null) {
          realObjectClass = object._class
        }
      })
    } else {
      query.unsubscribe()
    }
  }

  let oldClass: Ref<Class<Doc>>

  $: if (_class !== oldClass) {
    oldClass = _class
    mainEditor = undefined
  }

  let keys: KeyedAttribute[] = []
  let fieldEditors: Array<{ key: KeyedAttribute, editor: AnyComponent, category: AttributeCategory }> = []

  let mixins: Array<Mixin<Doc>> = []
  let showAllMixins = false

  $: mixins = getDocMixins(object, showAllMixins, ignoreMixins, realObjectClass)

  const dispatch = createEventDispatcher()

  let ignoreKeys: string[] = []
  let activityOptions = { enabled: true, showInput: true }
  let allowedCollections: string[] = []
  let collectionArrays: string[] = []
  let inplaceAttributes: string[] = []
  let ignoreMixins: Set<Ref<Mixin<Doc>>> = new Set<Ref<Mixin<Doc>>>()

  async function updateKeys (): Promise<void> {
    const keysMap = new Map(getFiltredKeys(hierarchy, realObjectClass, ignoreKeys).map((p) => [p.attr._id, p]))
    for (const m of mixins) {
      const mkeys = getFiltredKeys(hierarchy, m._id, ignoreKeys)
      for (const key of mkeys) {
        keysMap.set(key.attr._id, key)
      }
    }
    const filtredKeys = Array.from(keysMap.values())
    const { attributes, collections } = categorizeFields(hierarchy, filtredKeys, collectionArrays, allowedCollections)

    keys = attributes.map((it) => it.key)

    const editors: Array<{ key: KeyedAttribute, editor: AnyComponent, category: AttributeCategory }> = []
    const newInplaceAttributes: string[] = []

    for (const k of collections) {
      if (allowedCollections.includes(k.key.key)) continue
      const editor = await getFieldEditor(k.key)
      if (editor === undefined) continue
      if (k.category === 'inplace') {
        newInplaceAttributes.push(k.key.key)
      }
      editors.push({ key: k.key, editor, category: k.category })
    }
    inplaceAttributes = newInplaceAttributes
    fieldEditors = editors.sort((a, b) => AttributeCategoryOrder[a.category] - AttributeCategoryOrder[b.category])
  }

  interface MixinEditor {
    editor: AnyComponent
    pinned?: boolean
  }

  function getEditor (_class: Ref<Class<Doc>>): MixinEditor {
    const clazz = hierarchy.getClass(_class)
    const editorMixin = hierarchy.as(clazz, view.mixin.ObjectEditor)
    if (editorMixin?.editor == null && clazz.extends != null) return getEditor(clazz.extends)
    return { editor: editorMixin.editor, pinned: editorMixin?.pinned }
  }

  function getEditorFooter (
    _class: Ref<Class<Doc>>,
    object?: Doc
  ): { footer: AnyComponent, props?: Record<string, any> } | undefined {
    if (object !== undefined) {
      const footer = hierarchy.findClassOrMixinMixin(object, view.mixin.ObjectEditorFooter)
      if (footer !== undefined) {
        return { footer: footer.editor, props: footer.props }
      }
    }

    return undefined
  }

  let mainEditor: MixinEditor | undefined

  $: editorFooter = getEditorFooter(_class, object)

  $: getEditorOrDefault(realObjectClass, _id)

  async function getEditorOrDefault (_class: Ref<Class<Doc>>, _id: Ref<Doc>): Promise<void> {
    await updateKeys()
    mainEditor = getEditor(_class)
  }

  async function getFieldEditor (key: KeyedAttribute): Promise<AnyComponent | undefined> {
    const attrClass = getAttributePresenterClass(hierarchy, key.attr)
    const clazz = hierarchy.getClass(attrClass.attrClass)
    const mix = {
      array: view.mixin.ArrayEditor,
      collection: view.mixin.CollectionEditor,
      inplace: view.mixin.InlineAttributEditor,
      attribute: view.mixin.AttributeEditor,
      object: undefined
    }
    const mixinRef = mix[attrClass.category]
    if (mixinRef) {
      const editorMixin = hierarchy.as(clazz, mixinRef)
      return (editorMixin as any).editor
    } else {
      return undefined
    }
  }

  let title: string | undefined = undefined
  let rawTitle: string = ''

  $: if (object !== undefined) {
    getDocLabel(pClient, object).then((t) => {
      if (t) {
        rawTitle = t
      }
    })
  }

  async function getHeaderEditor (_class: Ref<Class<Doc>>): Promise<AnyComponent | undefined> {
    const editorMixin = hierarchy.classHierarchyMixin(
      _class,
      view.mixin.ObjectEditorHeader,
      (m) => hasResource(m.editor) ?? false
    )
    return editorMixin?.editor
  }

  let headerEditor: AnyComponent | undefined = undefined
  let headerLoading = false
  $: {
    headerLoading = true
    getHeaderEditor(realObjectClass).then((r) => {
      headerEditor = r
      headerLoading = false
    })
  }

  const _update = (result: any): void => {
    dispatch('update', result)
  }
  let panelWidth: number = 0
  let innerWidth: number = 0

  function handleOpen (ev: CustomEvent): void {
    ignoreKeys = ev.detail.ignoreKeys
    activityOptions = ev.detail.activityOptions ?? activityOptions
    ignoreMixins = new Set(ev.detail.ignoreMixins)
    allowedCollections = ev.detail.allowedCollections ?? []
    collectionArrays = ev.detail.collectionArrays ?? []
    title = ev.detail.title
    mixins = getDocMixins(object, showAllMixins, ignoreMixins, realObjectClass)
    updateKeys()
  }

  $: finalTitle = title ?? rawTitle
  let content: HTMLElement
</script>

{#if !embedded}
  <ActionContext
    context={{
      mode: 'editor'
    }}
  />
{/if}

{#if object !== undefined && finalTitle !== undefined}
  <Panel
    {object}
    isHeader={mainEditor?.pinned ?? false}
    allowClose={!embedded}
    isAside={true}
    {embedded}
    bind:content
    bind:panelWidth
    bind:innerWidth
    on:open
    on:update={(ev) => {
      _update(ev.detail)
    }}
    on:close={() => {
      dispatch('close')
    }}
    withoutActivity={!activityOptions.enabled}
    withoutInput={!activityOptions.showInput || readonly}
  >
    <svelte:fragment slot="title">
      {#if !embedded}<ParentsNavigator element={object} />{/if}
      {#if embedded && object}
        <DocNavLink noUnderline {object}>
          <div class="title">{finalTitle}</div>
        </DocNavLink>
      {:else}<div class="title not-active">{finalTitle}</div>{/if}
    </svelte:fragment>

    <svelte:fragment slot="utils">
      {#if !readonly}
        <Button
          icon={IconMoreH}
          iconProps={{ size: 'medium' }}
          kind={'icon'}
          on:click={(e) => {
            showMenu(e, { object, excludedActions: [view.action.Open] })
          }}
        />
      {/if}
      <Button
        icon={IconMixin}
        iconProps={{ size: 'medium' }}
        kind={'icon'}
        selected={showAllMixins}
        on:click={() => {
          showAllMixins = !showAllMixins
        }}
      />
    </svelte:fragment>

    <svelte:fragment slot="attributes" let:direction={dir}>
      {#if !headerLoading}
        {#if headerEditor !== undefined}
          <Component
            is={headerEditor}
            props={{
              object,
              keys,
              mixins,
              ignoreKeys,
              vertical: dir === 'column',
              allowedCollections,
              embedded,
              readonly
            }}
            on:update={updateKeys}
          />
        {:else if dir === 'column'}
          <DocAttributeBar
            {object}
            {mixins}
            {readonly}
            ignoreKeys={[...ignoreKeys, ...collectionArrays, ...inplaceAttributes]}
            {allowedCollections}
            on:update={updateKeys}
          />
        {:else}
          <AttributesBar {object} _class={realObjectClass} {keys} {readonly} />
        {/if}
      {/if}
    </svelte:fragment>

    <svelte:fragment slot="header">
      {#if mainEditor && mainEditor.editor && mainEditor.pinned}
        <div class="flex-col flex-grow my-4">
          <Component is={mainEditor.editor} props={{ object, readonly }} on:open={handleOpen} />
        </div>
      {/if}
    </svelte:fragment>

    {#if mainEditor && mainEditor.editor && !mainEditor.pinned}
      <div class="flex-col flex-grow flex-no-shrink step-tb-6">
        <Component is={mainEditor.editor} props={{ object, readonly }} on:open={handleOpen} />
      </div>
    {/if}

    {#each fieldEditors as collection}
      {#if collection.editor}
        <div class="step-tb-6">
          <Component
            is={collection.editor}
            props={{
              objectId: object._id,
              _class: collection.key.attr.attributeOf,
              object,
              space: object.space,
              key: collection.key,
              readonly,
              [collection.key.key]: getCollectionCounter(hierarchy, object, collection.key)
            }}
          />
        </div>
      {/if}
    {/each}

    {#if editorFooter}
      <div class="step-tb-6">
        <Component is={editorFooter.footer} props={{ object, _class, ...editorFooter.props, readonly }} />
      </div>
    {/if}
  </Panel>
{/if}
