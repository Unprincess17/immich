<script lang="ts" module>
  import { Protocol } from 'pmtiles';

  let protocol = new Protocol();
  void maplibregl.addProtocol('pmtiles', protocol.tile);
  void maplibregl.setRTLTextPlugin(mapboxRtlUrl, true);
</script>

<script lang="ts">
  import { afterNavigate } from '$app/navigation';
  import Icon from '$lib/components/elements/icon.svelte';
  import { Theme } from '$lib/constants';
  import { themeManager } from '$lib/managers/theme-manager.svelte';
  import MapSettingsModal from '$lib/modals/MapSettingsModal.svelte';
  import { mapSettings } from '$lib/stores/preferences.store';
  import { serverConfig } from '$lib/stores/server-config.store';
  import { getAssetThumbnailUrl, handlePromiseError } from '$lib/utils';
  import { getMapMarkers, type MapMarkerResponseDto } from '@immich/sdk';
  import { modalManager } from '@immich/ui';
  import mapboxRtlUrl from '@mapbox/mapbox-gl-rtl-text/mapbox-gl-rtl-text.min.js?url';
  import { mdiCog, mdiMap, mdiMapMarker } from '@mdi/js';
  import type { Feature, GeoJsonProperties, Geometry, Point } from 'geojson';
  import { isEqual, omit } from 'lodash-es';
  import { DateTime, Duration } from 'luxon';
  import maplibregl, { GlobeControl, type GeoJSONSource, type LngLatLike } from 'maplibre-gl';
  import { onDestroy, onMount, untrack } from 'svelte';
  import { t } from 'svelte-i18n';
  import {
    AttributionControl,
    Control,
    ControlButton,
    ControlGroup,
    FullscreenControl,
    GeoJSON,
    GeolocateControl,
    MapLibre,
    MarkerLayer,
    NavigationControl,
    Popup,
    ScaleControl,
    type Map,
  } from 'svelte-maplibre';

  interface Props {
    mapMarkers?: MapMarkerResponseDto[];
    showSettings?: boolean;
    zoom?: number | undefined;
    center?: LngLatLike | undefined;
    hash?: boolean;
    simplified?: boolean;
    clickable?: boolean;
    useLocationPin?: boolean;
    onOpenInMapView?: (() => Promise<void> | void) | undefined;
    onSelect?: (assetIds: string[]) => void;
    onClickPoint?: ({ lat, lng }: { lat: number; lng: number }) => void;
    popup?: import('svelte').Snippet<[{ marker: MapMarkerResponseDto }]>;
    rounded?: boolean;
    showSimpleControls?: boolean;
    autoFitBounds?: boolean;
    onViewportChange?: (viewport: { bounds: maplibregl.LngLatBounds; zoom: number }) => void;
    fullscreenContainer?: HTMLElement | string;
  }

  let {
    mapMarkers = $bindable(),
    showSettings = true,
    zoom = undefined,
    center = $bindable(undefined),
    hash = false,
    simplified = false,
    clickable = false,
    useLocationPin = false,
    onOpenInMapView = undefined,
    onSelect = () => {},
    onClickPoint = () => {},
    popup,
    rounded = false,
    showSimpleControls = true,
    autoFitBounds = true,
    onViewportChange = () => {},
    fullscreenContainer = undefined,
  }: Props = $props();

  // Calculate initial bounds from markers once during initialization
  const initialBounds = (() => {
    if (!autoFitBounds || center || zoom !== undefined || !mapMarkers || mapMarkers.length === 0) {
      return undefined;
    }

    const bounds = new maplibregl.LngLatBounds();
    for (const marker of mapMarkers) {
      bounds.extend([marker.lon, marker.lat]);
    }
    return bounds;
  })();

  let map: maplibregl.Map | undefined = $state();
  let marker: maplibregl.Marker | null = null;
  let abortController: AbortController;

  const theme = $derived($mapSettings.allowDarkMode ? themeManager.value : Theme.LIGHT);
  const styleUrl = $derived(theme === Theme.DARK ? $serverConfig.mapDarkStyleUrl : $serverConfig.mapLightStyleUrl);

  // Hidden layer ids to enable queryRenderedFeatures on GeoJSON source
  const QUERY_POINT_LAYER_ID = 'geojson-points-query';
  const QUERY_CLUSTER_LAYER_ID = 'geojson-clusters-query';

  // Bound based query for assets in viewport
  export function getAssetsInViewport(): string[] {
    if (!map || !mapMarkers) {
      return [];
    }

    try {
      const bounds = map.getBounds();
      const assetIds = mapMarkers
        .filter((marker) => bounds.contains([marker.lon, marker.lat]))
        .map((marker) => marker.id);

      return assetIds;
    } catch (error) {
      console.error('Error getting assets in viewport:', error);
      return [];
    }
  }

  // Rendered features based query for assets in viewport
  export async function getRenderedAssetsInViewport(): Promise<string[]> {
    if (!map) {
      return [];
    }

    try {
      const features = map.queryRenderedFeatures(undefined, {
        layers: [QUERY_POINT_LAYER_ID, QUERY_CLUSTER_LAYER_ID],
      });

      const assetIds: string[] = [];

      for (const feature of features) {
        if (feature.properties?.cluster_id) {
          const source = map.getSource('geojson') as GeoJSONSource;
          const leaves = await source.getClusterLeaves(feature.properties.cluster_id, Infinity, 0);
          for (const leaf of leaves) {
            const id = leaf.properties?.id;
            if (id) assetIds.push(id);
          }
        } else if (feature.properties?.id) {
          assetIds.push(feature.properties.id);
        }
      }

      return [...new Set(assetIds)];
    } catch (error) {
      console.error('Error getting rendered assets in viewport:', error);
      return [];
    }
  }

  export function addClipMapMarker(lng: number, lat: number) {
    if (map) {
      if (marker) {
        marker.remove();
      }

      center = { lng, lat };
      marker = new maplibregl.Marker().setLngLat([lng, lat]).addTo(map);
    }
  }

  function handleAssetClick(assetId: string, map: Map | null) {
    if (!map) {
      return;
    }
    onSelect([assetId]);
  }

  async function handleClusterClick(clusterId: number, map: Map | null) {
    if (!map) {
      return;
    }

    const mapSource = map?.getSource('geojson') as GeoJSONSource;
    const leaves = await mapSource.getClusterLeaves(clusterId, Infinity, 0);
    const ids = leaves.map((leaf) => leaf.properties?.id);
    onSelect(ids);
  }

  function handleMapClick(event: maplibregl.MapMouseEvent) {
    if (clickable) {
      const { lng, lat } = event.lngLat;
      onClickPoint({ lng, lat });

      if (marker) {
        marker.remove();
      }

      if (map) {
        marker = new maplibregl.Marker().setLngLat([lng, lat]).addTo(map);
      }
    }
  }

  function handleViewportChange(event: maplibregl.MapMouseEvent) {
    console.log('Viewport changed', event);
  }

  type FeaturePoint = Feature<Point, { id: string; city: string | null; state: string | null; country: string | null }>;

  const asFeature = (marker: MapMarkerResponseDto): FeaturePoint => {
    return {
      type: 'Feature',
      geometry: { type: 'Point', coordinates: [marker.lon, marker.lat] },
      properties: {
        id: marker.id,
        city: marker.city,
        state: marker.state,
        country: marker.country,
      },
    };
  };

  const asMarker = (feature: Feature<Geometry, GeoJsonProperties>): MapMarkerResponseDto => {
    const featurePoint = feature as FeaturePoint;
    const coords = maplibregl.LngLat.convert(featurePoint.geometry.coordinates as [number, number]);
    return {
      lat: coords.lat,
      lon: coords.lng,
      id: featurePoint.properties.id,
      city: featurePoint.properties.city,
      state: featurePoint.properties.state,
      country: featurePoint.properties.country,
    };
  };

  function getFileCreatedDates() {
    const { relativeDate, dateAfter, dateBefore } = $mapSettings;

    if (relativeDate) {
      const duration = Duration.fromISO(relativeDate);
      return {
        fileCreatedAfter: duration.isValid ? DateTime.now().minus(duration).toISO() : undefined,
      };
    }

    try {
      return {
        fileCreatedAfter: dateAfter ? new Date(dateAfter).toISOString() : undefined,
        fileCreatedBefore: dateBefore ? new Date(dateBefore).toISOString() : undefined,
      };
    } catch {
      $mapSettings.dateAfter = '';
      $mapSettings.dateBefore = '';
      return {};
    }
  }

  async function loadMapMarkers() {
    if (abortController) {
      abortController.abort();
    }
    abortController = new AbortController();

    const { includeArchived, onlyFavorites, withPartners, withSharedAlbums } = $mapSettings;
    const { fileCreatedAfter, fileCreatedBefore } = getFileCreatedDates();

    return await getMapMarkers(
      {
        isArchived: includeArchived || undefined,
        isFavorite: onlyFavorites || undefined,
        fileCreatedAfter: fileCreatedAfter || undefined,
        fileCreatedBefore,
        withPartners: withPartners || undefined,
        withSharedAlbums: withSharedAlbums || undefined,
      },
      {
        signal: abortController.signal,
      },
    );
  }

  const handleSettingsClick = async () => {
    const settings = await modalManager.show(MapSettingsModal, { settings: { ...$mapSettings } });
    if (settings) {
      const shouldUpdate = !isEqual(omit(settings, 'allowDarkMode'), omit($mapSettings, 'allowDarkMode'));
      $mapSettings = settings;

      if (shouldUpdate) {
        mapMarkers = await loadMapMarkers();
      }
    }
  };

  afterNavigate(() => {
    if (map) {
      map.resize();

      if (globalThis.location.hash) {
        const hashChangeEvent = new HashChangeEvent('hashchange');
        globalThis.dispatchEvent(hashChangeEvent);
      }
    }
  });

  onMount(async () => {
    if (!mapMarkers) {
      mapMarkers = await loadMapMarkers();
    }
  });

  onDestroy(() => {
    abortController?.abort();
  });

  $effect(() => {
    map?.setStyle(styleUrl, {
      transformStyle: (previousStyle, nextStyle) => {
        if (previousStyle) {
          // Preserves the custom map markers from the previous style when the theme is switched
          // Required until https://github.com/dimfeld/svelte-maplibre/issues/146 is fixed
          const customLayers = previousStyle.layers.filter((l) => l.type == 'fill' && l.source == 'geojson');
          const layers = nextStyle.layers.concat(customLayers);
          const sources = nextStyle.sources;

          for (const [key, value] of Object.entries(previousStyle.sources || {})) {
            if (key.startsWith('geojson')) {
              sources[key] = value;
            }
          }

          return {
            ...nextStyle,
            sources,
            layers,
          };
        }
        return nextStyle;
      },
    });
  });

  $effect(() => {
    if (!center || !zoom) {
      return;
    }

    untrack(() => map?.jumpTo({ center, zoom }));
  });
</script>

<!--  We handle style loading ourselves so we set style blank here -->
<MapLibre
  {hash}
  style=""
  class="h-full {rounded ? 'rounded-2xl' : 'rounded-none'}"
  {zoom}
  {center}
  bounds={initialBounds}
  fitBoundsOptions={{ padding: 50, maxZoom: 15 }}
  attributionControl={false}
  diffStyleUpdates={true}
  onload={(event) => {
    event.setMaxZoom(18);
    event.on('click', handleMapClick);

    // Add hidden layers to enable queryRenderedFeatures on the GeoJSON source
    try {
      if (!event.getLayer(QUERY_POINT_LAYER_ID)) {
        event.addLayer({
          id: QUERY_POINT_LAYER_ID,
          type: 'circle',
          source: 'geojson',
          filter: ['!', ['has', 'point_count']],
          paint: { 'circle-radius': 1, 'circle-opacity': 0 },
        });
      }
      if (!event.getLayer(QUERY_CLUSTER_LAYER_ID)) {
        event.addLayer({
          id: QUERY_CLUSTER_LAYER_ID,
          type: 'circle',
          source: 'geojson',
          filter: ['has', 'point_count'],
          paint: { 'circle-radius': 1, 'circle-opacity': 0 },
        });
      }
    } catch (e) {
      console.warn('Unable to add query layers for rendered-features queries', e);
    }

    event.on('moveend', () => {
      const bounds = event.getBounds();
      const zoom = event.getZoom();
      onViewportChange({ bounds, zoom });
    });
    event.on('zoomend', () => {
      const bounds = event.getBounds();
      const zoom = event.getZoom();
      onViewportChange({ bounds, zoom });
    });

    if (!simplified) {
      event.addControl(new GlobeControl(), 'top-left');
    }
  }}
  bind:map
>
  {#snippet children({ map }: { map: maplibregl.Map })}
    {#if showSimpleControls}
      <NavigationControl position="top-left" showCompass={!simplified} />

      {#if !simplified}
        <GeolocateControl position="top-left" />
        <FullscreenControl position="top-left" container={fullscreenContainer} />
        <ScaleControl />
        <AttributionControl compact={false} />
      {/if}
    {/if}

    {#if showSettings}
      <Control>
        <ControlGroup>
          <ControlButton onclick={handleSettingsClick}
            ><Icon path={mdiCog} size="100%" class="text-black/80" /></ControlButton
          >
        </ControlGroup>
      </Control>
    {/if}

    {#if onOpenInMapView && showSimpleControls}
      <Control position="top-right">
        <ControlGroup>
          <ControlButton onclick={() => onOpenInMapView()}>
            <Icon title={$t('open_in_map_view')} path={mdiMap} size="100%" class="text-black/80" />
          </ControlButton>
        </ControlGroup>
      </Control>
    {/if}

    <GeoJSON
      data={{
        type: 'FeatureCollection',
        features: mapMarkers?.map((marker) => asFeature(marker)) ?? [],
      }}
      id="geojson"
      cluster={{ radius: 35, maxZoom: 18 }}
    >
      <MarkerLayer
        applyToClusters
        asButton
        onclick={(event) => handlePromiseError(handleClusterClick(event.feature.properties?.cluster_id, map))}
      >
        {#snippet children({ feature })}
          <div
            class="rounded-full w-[40px] h-[40px] bg-immich-primary text-white flex justify-center items-center font-mono font-bold shadow-lg hover:bg-immich-dark-primary transition-all duration-200 hover:text-immich-dark-bg opacity-90"
          >
            {feature.properties?.point_count}
          </div>
        {/snippet}
      </MarkerLayer>
      <MarkerLayer
        applyToClusters={false}
        asButton
        onclick={(event) => {
          if (!popup) {
            handleAssetClick(event.feature.properties?.id, map);
          }
        }}
      >
        {#snippet children({ feature }: { feature: Feature<Geometry, GeoJsonProperties> })}
          {#if useLocationPin}
            <Icon
              path={mdiMapMarker}
              size="50px"
              class="dark:text-immich-dark-primary text-immich-primary -translate-y-[50%]"
            />
          {:else}
            <img
              src={getAssetThumbnailUrl(feature.properties?.id)}
              class="rounded-full w-[60px] h-[60px] border-2 border-immich-primary shadow-lg hover:border-immich-dark-primary transition-all duration-200 hover:scale-150 object-cover bg-immich-primary"
              alt={feature.properties?.city && feature.properties.country
                ? $t('map_marker_for_images', {
                    values: { city: feature.properties.city, country: feature.properties.country },
                  })
                : $t('map_marker_with_image')}
            />
          {/if}
          {#if popup}
            <Popup offset={[0, -30]} openOn="click" closeOnClickOutside>
              {@render popup?.({ marker: asMarker(feature) })}
            </Popup>
          {/if}
        {/snippet}
      </MarkerLayer>
    </GeoJSON>
  {/snippet}
</MapLibre>
