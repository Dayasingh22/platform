<!--
// Copyright © 2022 Hardcore Engineering Inc.
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
  import { Attachment } from '@hcengineering/attachment'
  import core, { Account, Class, Doc, IdMap, Markup, Ref, Space, generateId, toIdMap } from '@hcengineering/core'
  import { Asset, IntlString, setPlatformStatus, unknownError } from '@hcengineering/platform'
  import {
    DraftController,
    createQuery,
    deleteFile,
    draftsStore,
    getClient,
    getFileMetadata,
    uploadFile
  } from '@hcengineering/presentation'
  import textEditor, { AttachIcon, EmptyMarkup, ReferenceInput, type RefAction } from '@hcengineering/text-editor'
  import { Loading, type AnySvelteComponent } from '@hcengineering/ui'
  import { createEventDispatcher, onDestroy, tick } from 'svelte'
  import attachment from '../plugin'
  import AttachmentPresenter from './AttachmentPresenter.svelte'

  export let objectId: Ref<Doc>
  export let space: Ref<Space>
  export let _class: Ref<Class<Doc>>
  export let content: Markup = EmptyMarkup
  export let iconSend: Asset | AnySvelteComponent | undefined = undefined
  export let labelSend: IntlString | undefined = undefined
  export let showSend = true
  export let showActions = true
  export let shouldSaveDraft: boolean = false
  export let attachments: IdMap<Attachment> = new Map()
  export let loading = false
  export let focusIndex: number = -1
  export let autofocus = false
  export function submit (): void {
    refInput.submit()
  }
  export let placeholder: IntlString | undefined = undefined
  export let extraActions: RefAction[] = []
  export let boundary: HTMLElement | undefined = undefined

  let refInput: ReferenceInput

  let inputFile: HTMLInputElement
  let saved = false
  const dispatch = createEventDispatcher()

  const client = getClient()
  const query = createQuery()

  const draftKey = `${objectId}_attachments`
  const draftController = new DraftController<Record<Ref<Attachment>, Attachment>>(draftKey)

  let draftAttachments: Record<Ref<Attachment>, Attachment> | undefined = undefined
  let originalAttachments: Set<Ref<Attachment>> = new Set<Ref<Attachment>>()
  const newAttachments: Set<Ref<Attachment>> = new Set<Ref<Attachment>>()
  const removedAttachments: Set<Attachment> = new Set<Attachment>()

  let progress = false

  let refContainer: HTMLElement

  $: objectId && updateAttachments(objectId)

  async function updateAttachments (objectId: Ref<Doc>) {
    draftAttachments = $draftsStore[draftKey]
    if (draftAttachments && shouldSaveDraft) {
      attachments.clear()
      newAttachments.clear()
      Object.entries(draftAttachments).map((file) => {
        return attachments.set(file[0] as Ref<Attachment>, file[1])
      })
      Object.entries(draftAttachments).map((file) => {
        return newAttachments.add(file[0] as Ref<Attachment>)
      })
      originalAttachments.clear()
      removedAttachments.clear()
    } else {
      query.query(
        attachment.class.Attachment,
        {
          attachedTo: objectId
        },
        (res) => {
          originalAttachments = new Set(res.map((p) => p._id))
          attachments = toIdMap(res)
        },
        {
          lookup: {
            file: core.class.Blob
          }
        }
      )
    }
  }

  async function saveDraft () {
    if (shouldSaveDraft) {
      draftAttachments = Object.fromEntries(attachments)
      draftController.save(draftAttachments)
    }
  }

  async function createAttachment (file: File) {
    try {
      const uuid = await uploadFile(file)
      const metadata = await getFileMetadata(file, uuid)

      const _id: Ref<Attachment> = generateId()
      attachments.set(_id, {
        _id,
        _class: attachment.class.Attachment,
        collection: 'attachments',
        modifiedOn: 0,
        modifiedBy: '' as Ref<Account>,
        space,
        attachedTo: objectId,
        attachedToClass: _class,
        name: file.name,
        file: uuid,
        type: file.type,
        size: file.size,
        lastModified: file.lastModified,
        metadata
      })
      newAttachments.add(_id)
      attachments = attachments
      saveDraft()
    } catch (err: any) {
      setPlatformStatus(unknownError(err))
    }
  }

  const existingAttachmentsQuery = createQuery()
  let existingAttachments: Ref<Attachment>[] = []
  $: existingAttachmentsQuery.query(
    attachment.class.Attachment,
    {
      space,
      attachedTo: objectId,
      attachedToClass: _class,
      _id: { $in: Array.from(attachments.keys()) }
    },
    (res) => {
      existingAttachments = res.map((p) => p._id)
    }
  )

  async function saveAttachment (doc: Attachment): Promise<void> {
    if (!existingAttachments.includes(doc._id)) {
      await client.addCollection(attachment.class.Attachment, space, objectId, _class, 'attachments', doc, doc._id)
    }
  }

  async function fileSelected (): Promise<void> {
    progress = true
    await tick()
    const list = inputFile.files
    if (list === null || list.length === 0) return
    for (let index = 0; index < list.length; index++) {
      const file = list.item(index)
      if (file !== null) {
        await createAttachment(file)
      }
    }
    inputFile.value = ''
    progress = false
  }

  async function fileDrop (e: DragEvent): Promise<void> {
    progress = true
    const list = e.dataTransfer?.files
    if (list === undefined || list.length === 0) return
    for (let index = 0; index < list.length; index++) {
      const file = list.item(index)
      if (file !== null) {
        await createAttachment(file)
      }
    }
    progress = false
  }

  async function removeAttachment (attachment: Attachment): Promise<void> {
    removedAttachments.add(attachment)
    attachments.delete(attachment._id)
    if (shouldSaveDraft) {
      await createAttachments()
    }
    attachments = attachments
    saveDraft()
  }

  async function deleteAttachment (attachment: Attachment): Promise<void> {
    if (originalAttachments.has(attachment._id)) {
      await client.removeCollection(
        attachment._class,
        attachment.space,
        attachment._id,
        attachment.attachedTo,
        attachment.attachedToClass,
        'attachments'
      )
    } else {
      await deleteFile(attachment.file)
    }
  }

  onDestroy(() => {
    if (!saved && !shouldSaveDraft) {
      newAttachments.forEach((p) => {
        const attachment = attachments.get(p)
        if (attachment !== undefined) {
          void deleteAttachment(attachment)
        }
      })
    }
    if (!saved && shouldSaveDraft) {
      void createAttachments()
    }
  })

  export function removeDraft (removeFiles: boolean) {
    draftController.remove()
    if (removeFiles) {
      newAttachments.forEach((p) => {
        const attachment = attachments.get(p)
        if (attachment !== undefined) {
          void deleteFile(attachment.file)
        }
      })
    }
  }

  export function createAttachments (): Promise<void> {
    saved = true
    const promises: Promise<any>[] = []
    newAttachments.forEach((p) => {
      const attachment = attachments.get(p)
      if (attachment !== undefined) {
        promises.push(saveAttachment(attachment))
      }
    })
    removedAttachments.forEach((p) => {
      promises.push(deleteAttachment(p))
    })
    return Promise.all(promises).then()
  }

  async function onMessage (event: CustomEvent) {
    loading = true
    await createAttachments()
    loading = false
    dispatch('message', { message: event.detail, attachments: attachments.size })
  }

  async function onUpdate (event: CustomEvent) {
    dispatch('update', { message: event.detail, attachments: attachments.size })
  }

  async function loadFiles (evt: ClipboardEvent): Promise<void> {
    progress = true
    const files = (evt.clipboardData?.files ?? []) as File[]

    for (const file of files) {
      await createAttachment(file)
    }

    progress = false
  }

  function pasteAction (_: any, evt: ClipboardEvent): boolean {
    let target: HTMLElement | null = evt.target as HTMLElement
    let allowed = false

    while (target != null) {
      target = target.parentElement
      if (target === refContainer) {
        allowed = true
      }
    }
    if (!allowed) {
      return false
    }

    const hasFiles = Array.from(evt.clipboardData?.items ?? []).some((i) => i.kind === 'file')

    if (!hasFiles) {
      return false
    }

    void loadFiles(evt)
    return true
  }
</script>

<div class="no-print" bind:this={refContainer}>
  <input
    bind:this={inputFile}
    disabled={inputFile == null}
    multiple
    type="file"
    name="file"
    id="file"
    style="display: none"
    on:change={fileSelected}
  />
  <!-- svelte-ignore a11y-no-static-element-interactions -->
  <div
    class="container"
    on:dragover|preventDefault={() => {}}
    on:dragleave={() => {}}
    on:drop|preventDefault|stopPropagation={fileDrop}
  >
    <ReferenceInput
      {focusIndex}
      bind:this={refInput}
      {content}
      {iconSend}
      {labelSend}
      {showSend}
      {showActions}
      autofocus={autofocus ? 'end' : false}
      loading={loading || progress}
      {boundary}
      canEmbedFiles={false}
      canEmbedImages={false}
      extraActions={[
        ...extraActions,
        {
          label: textEditor.string.Attach,
          icon: AttachIcon,
          action: () => {
            dispatch('focus')
            inputFile.click()
          },
          order: 1001
        }
      ]}
      showHeader={attachments.size > 0 || progress}
      haveAttachment={attachments.size > 0}
      on:focus
      on:blur
      on:message={onMessage}
      on:update={onUpdate}
      onPaste={pasteAction}
      {placeholder}
    >
      <div slot="header">
        {#if attachments.size || progress}
          <div class="flex-row-center list scroll-divider-color">
            {#if progress}
              <div class="flex p-3">
                <Loading />
              </div>
            {/if}
            {#each Array.from(attachments.values()) as attachment}
              <div class="item flex">
                <AttachmentPresenter
                  value={attachment}
                  removable
                  on:remove={(result) => {
                    if (result !== undefined) removeAttachment(attachment)
                  }}
                />
              </div>
            {/each}
          </div>
        {/if}
      </div>
    </ReferenceInput>
  </div>
</div>

<style lang="scss">
  .list {
    padding: 0.5rem;
    overflow-x: auto;
    overflow-y: hidden;

    .item + .item {
      padding-left: 1rem;
      border-left: 1px solid var(--theme-divider-color);
    }
  }
</style>
