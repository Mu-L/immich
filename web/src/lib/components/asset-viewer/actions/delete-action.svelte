<script lang="ts">
  import { shortcuts } from '$lib/actions/shortcut';
  import DeleteAssetDialog from '$lib/components/photos-page/delete-asset-dialog.svelte';
  import {
    NotificationType,
    notificationController,
  } from '$lib/components/shared-components/notification/notification';
  import Portal from '$lib/components/shared-components/portal/portal.svelte';
  import { AssetAction } from '$lib/constants';
  import { showDeleteModal } from '$lib/stores/preferences.store';
  import { featureFlags } from '$lib/stores/server-config.store';
  import { handleError } from '$lib/utils/handle-error';
  import { toTimelineAsset } from '$lib/utils/timeline-util';
  import { deleteAssets, type AssetResponseDto } from '@immich/sdk';
  import { mdiDeleteForeverOutline, mdiDeleteOutline } from '@mdi/js';
  import { t } from 'svelte-i18n';
  import type { OnAction, PreAction } from './action';
  import { IconButton } from '@immich/ui';

  interface Props {
    asset: AssetResponseDto;
    onAction: OnAction;
    preAction: PreAction;
  }

  let { asset, onAction, preAction }: Props = $props();

  let showConfirmModal = $state(false);

  const trashOrDelete = async (force = false) => {
    if (force || !$featureFlags.trash) {
      if ($showDeleteModal) {
        showConfirmModal = true;
        return;
      }
      await deleteAsset();
      return;
    }

    await trashAsset();
    return;
  };

  const trashAsset = async () => {
    try {
      preAction({ type: AssetAction.TRASH, asset: toTimelineAsset(asset) });
      await deleteAssets({ assetBulkDeleteDto: { ids: [asset.id] } });
      onAction({ type: AssetAction.TRASH, asset: toTimelineAsset(asset) });

      notificationController.show({
        message: $t('moved_to_trash'),
        type: NotificationType.Info,
      });
    } catch (error) {
      handleError(error, $t('errors.unable_to_trash_asset'));
    }
  };

  const deleteAsset = async () => {
    try {
      preAction({ type: AssetAction.DELETE, asset: toTimelineAsset(asset) });
      await deleteAssets({ assetBulkDeleteDto: { ids: [asset.id], force: true } });
      onAction({ type: AssetAction.DELETE, asset: toTimelineAsset(asset) });

      notificationController.show({
        message: $t('permanently_deleted_asset'),
        type: NotificationType.Info,
      });
    } catch (error) {
      handleError(error, $t('errors.unable_to_delete_asset'));
    } finally {
      showConfirmModal = false;
    }
  };
</script>

<svelte:document
  use:shortcuts={[
    { shortcut: { key: 'Delete' }, onShortcut: () => trashOrDelete(asset.isTrashed) },
    { shortcut: { key: 'Delete', shift: true }, onShortcut: () => trashOrDelete(true) },
  ]}
/>

<IconButton
  color="secondary"
  shape="round"
  variant="ghost"
  icon={asset.isTrashed ? mdiDeleteForeverOutline : mdiDeleteOutline}
  aria-label={asset.isTrashed ? $t('permanently_delete') : $t('delete')}
  onclick={() => trashOrDelete(asset.isTrashed)}
/>

{#if showConfirmModal}
  <Portal target="body">
    <DeleteAssetDialog size={1} onCancel={() => (showConfirmModal = false)} onConfirm={deleteAsset} />
  </Portal>
{/if}
