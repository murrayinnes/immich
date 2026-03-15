<script lang="ts">
  import { browser } from '$app/environment';
  import { goto } from '$app/navigation';
  import OnEvents from '$lib/components/OnEvents.svelte';
  import UserPageLayout, { headerId } from '$lib/components/layouts/user-page-layout.svelte';
  import ButtonContextMenu from '$lib/components/shared-components/context-menu/button-context-menu.svelte';
  import GalleryViewer from '$lib/components/shared-components/gallery-viewer/gallery-viewer.svelte';
  import Breadcrumbs from '$lib/components/shared-components/tree/breadcrumbs.svelte';
  import TreeItemThumbnails from '$lib/components/shared-components/tree/tree-item-thumbnails.svelte';
  import TreeItems from '$lib/components/shared-components/tree/tree-items.svelte';
  import Sidebar from '$lib/components/sidebar/sidebar.svelte';
  import ArchiveAction from '$lib/components/timeline/actions/ArchiveAction.svelte';
  import ChangeDate from '$lib/components/timeline/actions/ChangeDateAction.svelte';
  import ChangeDescription from '$lib/components/timeline/actions/ChangeDescriptionAction.svelte';
  import ChangeLocation from '$lib/components/timeline/actions/ChangeLocationAction.svelte';
  import CreateSharedLink from '$lib/components/timeline/actions/CreateSharedLinkAction.svelte';
  import DeleteAssets from '$lib/components/timeline/actions/DeleteAssetsAction.svelte';
  import DownloadAction from '$lib/components/timeline/actions/DownloadAction.svelte';
  import FavoriteAction from '$lib/components/timeline/actions/FavoriteAction.svelte';
  import SetVisibilityAction from '$lib/components/timeline/actions/SetVisibilityAction.svelte';
  import TagAction from '$lib/components/timeline/actions/TagAction.svelte';
  import AssetSelectControlBar from '$lib/components/timeline/AssetSelectControlBar.svelte';
  import SkipLink from '$lib/elements/SkipLink.svelte';
  import type { Viewport } from '$lib/managers/timeline-manager/types';
  import { Route } from '$lib/route';
  import { getAssetBulkActions } from '$lib/services/asset.service';
  import { getTagActions } from '$lib/services/tag.service';
  import { AssetInteraction } from '$lib/stores/asset-interaction.svelte';
  import { preferences } from '$lib/stores/user.store';
  import { cancelMultiselect } from '$lib/utils/asset-utils';
  import { toTimelineAsset } from '$lib/utils/timeline-util';
  import { joinPaths, TreeNode } from '$lib/utils/tree-utils';
  import { AssetMediaSize, getAllTags, searchAssets, type AssetResponseDto, type TagResponseDto } from '@immich/sdk';
  import { ActionButton, CommandPaletteDefaultProvider, Icon, IconButton, Text } from '@immich/ui';
  import { mdiDotsVertical, mdiMagnify, mdiSelectAll, mdiTag, mdiTagMultiple } from '@mdi/js';
  import { t } from 'svelte-i18n';
  import type { PageData } from './$types';

  interface Props {
    data: PageData;
  }

  let { data }: Props = $props();

  const viewport: Viewport = $state({ width: 0, height: 0 });
  const assetInteraction = new AssetInteraction();

  let tags = $derived<TagResponseDto[]>(data.tags);
  const tree = $derived(TreeNode.fromTags(tags));
  const tag = $derived(tree.traverse(data.path));

  let assets = $state<AssetResponseDto[]>([]);
  let loading = $state(false);

  async function loadAssetsForTag(tagId: string): Promise<AssetResponseDto[]> {
    const all: AssetResponseDto[] = [];
    let page = 1;
    while (true) {
      const response = await searchAssets({ metadataSearchDto: { tagIds: [tagId], page, size: 1000 } });
      all.push(...response.assets.items);
      if (!response.assets.nextPage) break;
      page++;
    }
    return all;
  }

  $effect(() => {
    const tagId = tag?.id;
    if (tagId) {
      loading = true;
      assets = [];
      loadAssetsForTag(tagId)
        .then((result) => {
          assets = result;
        })
        .catch((error) => {
          console.error('Failed to load tag assets:', error);
        })
        .finally(() => {
          loading = false;
        });
    } else {
      assets = [];
    }
  });

  // Zoom
  const ZOOM_KEY = 'tags-zoom-level';
  const DEFAULT_ZOOM = 235;
  let zoomLevel = $state(browser ? Number(localStorage.getItem(ZOOM_KEY) || DEFAULT_ZOOM) : DEFAULT_ZOOM);
  $effect(() => {
    if (browser) localStorage.setItem(ZOOM_KEY, String(zoomLevel));
  });

  const effectiveWidth = $derived(viewport.width > 0 ? viewport.width : browser ? window.innerWidth : DEFAULT_ZOOM);
  const maxNativeHeight = $derived(
    assets.length > 0 ? Math.max(...assets.map((a) => a.exifInfo?.exifImageHeight ?? 0)) : 0,
  );
  const maxZoom = $derived(Math.min(effectiveWidth, maxNativeHeight || effectiveWidth));

  // Sort
  type SortKey = 'title' | 'date';
  const SORT_KEY = 'tags-sort';
  const DEFAULT_SORT: SortKey = 'title';
  let sortBy = $state<SortKey>(browser ? ((localStorage.getItem(SORT_KEY) as SortKey) ?? DEFAULT_SORT) : DEFAULT_SORT);
  $effect(() => {
    if (browser) localStorage.setItem(SORT_KEY, sortBy);
  });

  const sortedAssets = $derived.by(() => {
    const list = [...assets];
    switch (sortBy) {
      case 'date':
        return list.sort((a, b) => a.fileCreatedAt.localeCompare(b.fileCreatedAt));
      default:
        return list.sort((a, b) =>
          (a.exifInfo?.description ?? a.originalFileName).localeCompare(
            b.exifInfo?.description ?? b.originalFileName,
          ),
        );
    }
  });

  const handleNavigation = (tag: string) => navigateToView(joinPaths(data.path, tag));
  const getLink = (path: string) => Route.tags({ path });
  const navigateToView = (path: string) => goto(getLink(path));

  function handleSelectAllAssets() {
    assetInteraction.selectAssets(sortedAssets.map((a) => toTimelineAsset(a)));
  }

  const handleSetVisibility = (assetIds: string[]) => {
    assets = assets.filter((a) => !assetIds.includes(a.id));
    assetInteraction.clearMultiselect();
  };

  async function triggerAssetUpdate() {
    cancelMultiselect(assetInteraction);
    if (tag?.id) {
      assets = await loadAssetsForTag(tag.id);
    }
  }

  const onRefresh = async () => {
    tags = await getAllTags();
  };

  const onTagDelete = async (response: TreeNode) => {
    if (response.path === tag.path) {
      await navigateToView(tag.parent ? tag.parent.path : '');
    }
    await onRefresh();
  };

  const { Create, Update, Delete } = $derived(getTagActions($t, tag));
</script>

<OnEvents onTagCreate={onRefresh} onTagUpdate={onRefresh} {onTagDelete} />

<UserPageLayout title={data.meta.title} actions={[Create, Update, Delete]}>
  {#snippet sidebar()}
    <Sidebar>
      <SkipLink target={`#${headerId}`} text={$t('skip_to_tags')} breakpoint="md" />
      <section>
        <Text class="ps-4 mb-4" size="small">{$t('explorer')}</Text>
        <div class="h-full">
          <TreeItems icons={{ default: mdiTag, active: mdiTag }} {tree} active={tag.path} {getLink} />
        </div>
      </section>
    </Sidebar>
  {/snippet}

  <Breadcrumbs node={tag} icon={mdiTagMultiple} title={$t('tags')} {getLink}>
    {#snippet extra()}
      <div class="flex items-center gap-3 pe-1">
        <select
          bind:value={sortBy}
          class="text-xs bg-transparent border border-gray-200 dark:border-gray-700 rounded-lg px-2 py-1 text-gray-600 dark:text-gray-300 cursor-pointer focus:outline-none"
        >
          <option value="title">Title</option>
          <option value="date">Date</option>
        </select>
        <Icon icon={mdiMagnify} class="text-gray-500 dark:text-gray-300 shrink-0" size="16" />
        <input
          type="range"
          min={DEFAULT_ZOOM}
          max={maxZoom}
          step={5}
          bind:value={zoomLevel}
          class="w-32 cursor-pointer accent-immich-primary"
          aria-label={$t('zoom')}
        />
      </div>
    {/snippet}
  </Breadcrumbs>

  <section class="mt-2 h-[calc(100%-(--spacing(20)))] overflow-auto immich-scrollbar">
    {#if tag.hasAssets}
      {#if loading}
        <div class="flex justify-center items-center h-32 text-gray-400">{$t('loading')}</div>
      {:else if sortedAssets.length > 0}
        <div bind:clientHeight={viewport.height} bind:clientWidth={viewport.width}>
          <GalleryViewer
            assets={sortedAssets}
            {assetInteraction}
            {viewport}
            showAssetName={false}
            pageHeaderOffset={54}
            onReload={triggerAssetUpdate}
            rowHeight={zoomLevel}
            assetSize={AssetMediaSize.Preview}
          />
        </div>
      {:else}
        <TreeItemThumbnails items={tag.children} icon={mdiTag} onClick={handleNavigation} />
      {/if}
    {:else}
      <TreeItemThumbnails items={tag.children} icon={mdiTag} onClick={handleNavigation} />
    {/if}
  </section>
</UserPageLayout>

<section>
  {#if assetInteraction.selectionActive}
    <div class="fixed top-0 start-0 w-full">
      <AssetSelectControlBar
        assets={assetInteraction.selectedAssets}
        clearSelect={() => assetInteraction.clearMultiselect()}
      >
        {@const Actions = getAssetBulkActions($t, assetInteraction.asControlContext())}
        <CommandPaletteDefaultProvider name={$t('assets')} actions={Object.values(Actions)} />
        <CreateSharedLink />
        <IconButton
          shape="round"
          color="secondary"
          variant="ghost"
          aria-label={$t('select_all')}
          icon={mdiSelectAll}
          onclick={handleSelectAllAssets}
        />
        <ActionButton action={Actions.AddToAlbum} />
        <FavoriteAction
          removeFavorite={assetInteraction.isAllFavorite}
          onFavorite={(ids, isFavorite) => {
            for (const id of ids) {
              const asset = assets.find((a) => a.id === id);
              if (asset) asset.isFavorite = isFavorite;
            }
          }}
        />
        <ButtonContextMenu icon={mdiDotsVertical} title={$t('menu')}>
          <DownloadAction menuItem />
          <ChangeDate menuItem />
          <ChangeDescription menuItem />
          <ChangeLocation menuItem />
          <ArchiveAction menuItem unarchive={assetInteraction.isAllArchived} onArchive={triggerAssetUpdate} />
          {#if $preferences.tags.enabled}
            <TagAction menuItem />
          {/if}
          <DeleteAssets
            menuItem
            onAssetDelete={(assetIds) => {
              assets = assets.filter((a) => !assetIds.includes(a.id));
              assetInteraction.clearMultiselect();
            }}
            onUndoDelete={triggerAssetUpdate}
          />
          <SetVisibilityAction menuItem onVisibilitySet={handleSetVisibility} />
        </ButtonContextMenu>
      </AssetSelectControlBar>
    </div>
  {/if}
</section>
