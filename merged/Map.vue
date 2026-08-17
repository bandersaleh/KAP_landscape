<!-- // If 3D Map Functions in this file get updated, run powershell command: docker compose --profile local run --rm --entrypoint corepack node pnpm run build -->
<template>
  <div
    class="map-component-wrap"
    :class="{ 'is-map-expanded': isMapExpanded }"
    :style="mapExpandedStyle"
  >
    <div
      id="map"
      ref="map"
      class="h-500"
      :class="{ 'is-hidden-for-three': isStandaloneThreeMode }"
    ></div>

    <div
      v-show="isStandaloneThreeMode"
      ref="threeParkHost"
      class="three-park-host"
      aria-label="3D park terrain model"
    >
      <div v-if="threeParkLoading" class="three-park-status" role="status">
        {{
          mapUiText(
            "Building 3D terrain and GIS layers",
            "جارٍ إنشاء التضاريس وطبقات نظم المعلومات الجغرافية ثلاثية الأبعاد",
            "Building 3D terrain and GIS layers",
          )
        }}
      </div>
      <div
        v-else-if="threeParkError"
        class="three-park-status three-park-status--error"
        role="alert"
      >
        {{ threeParkError }}
      </div>
      <div
        v-else
        class="three-park-attribution"
        aria-label="Map data attribution"
      >
        © Mapbox © OpenStreetMap
      </div>

      <div
        v-if="!threeParkLoading && !threeParkError"
        class="three-park-compass"
        role="img"
        :aria-label="
          mapUiText(
            '3D map compass. North rotates with the current viewing direction.',
            'بوصلة الخريطة ثلاثية الأبعاد. يدور اتجاه الشمال حسب زاوية العرض الحالية.',
            '3D map compass. North rotates with the current viewing direction.',
          )
        "
      >
        <div class="three-park-compass__dial" aria-hidden="true">
          <div ref="threeCompassRose" class="three-park-compass__rose">
            <span class="three-park-compass__north">N</span>
            <svg
              class="three-park-compass__arrow"
              viewBox="0 0 44 52"
              focusable="false"
              aria-hidden="true"
            >
              <path
                class="three-park-compass__arrow-main"
                d="M22 1 L39 47 L22 38 L5 47 Z"
              />
              <path
                class="three-park-compass__arrow-cut"
                d="M22 12 L22 38 L13 43 Z"
              />
            </svg>
          </div>
          <span
            class="three-park-compass__center-dot"
            aria-hidden="true"
          ></span>
        </div>
      </div>

      <div
        v-if="
          !threeParkLoading && !threeParkError && threeToolState.structureLabels
        "
        class="three-park-label-overlay"
        aria-hidden="true"
      >
        <div
          v-for="annotation in threeVisibleStructureLabels"
          :key="annotation.id"
          class="three-park-label-overlay__item"
          :style="{
            left: `${annotation.x}px`,
            top: `${annotation.y}px`,
          }"
        >
          {{ annotation.text }}
        </div>
      </div>
    </div>

    <div
      class="main-map-style-switch"
      :class="{ 'is-dark-mode': theme === 'dark', 'is-arabic': isArabicUI() }"
      :dir="isArabicUI() ? 'rtl' : 'ltr'"
      :aria-label="
        mapUiText(
          'Basemap style controls',
          'أزرار نمط الخريطة',
          'Basemap style controls',
        )
      "
    >
      <button
        v-for="mode in visibleBaseMapModes"
        :key="mode.value"
        type="button"
        class="main-map-style-switch__button"
        :class="{ 'is-active': activeBaseMapMode === mode.value }"
        :aria-pressed="activeBaseMapMode === mode.value ? 'true' : 'false'"
        :title="mapUiLabel(mode)"
        :data-map-mode="mode.value"
        @click="setBaseMapMode(mode.value)"
      >
        {{ mapUiLabel(mode) }}
      </button>
    </div>

    <section
      v-if="isStandaloneThreeMode && threeToolState.treeTypes"
      class="three-tree-type-panel"
      :class="{ 'is-arabic': isArabicUI() }"
      :dir="isArabicUI() ? 'rtl' : 'ltr'"
      :aria-label="mapUiText('Tree Type', 'أنواع الأشجار', 'Tree Type')"
    >
      <div class="three-tree-type-panel__header">
        <div>
          <h3 class="three-tree-type-panel__title">
            {{ mapUiText("Tree Type", "أنواع الأشجار", "Tree Type") }}
          </h3>
          <p class="three-tree-type-panel__total">
            {{ mapUiText("Total Trees", "إجمالي الأشجار", "Total Trees") }}:
            <strong>{{ threeTreeTypeTotal }}</strong>
          </p>
        </div>
        <button
          type="button"
          class="three-tree-type-panel__close"
          :aria-label="
            mapUiText(
              'Close Tree Type',
              'إغلاق أنواع الأشجار',
              'Close Tree Type',
            )
          "
          @click="$set(threeToolState, 'treeTypes', false)"
        >
          ×
        </button>
      </div>

      <div v-if="threeTreeTypeLoading" class="three-tree-type-panel__message">
        {{
          mapUiText(
            "Loading tree types…",
            "جارٍ تحميل أنواع الأشجار…",
            "Loading tree types…",
          )
        }}
      </div>
      <div
        v-else-if="!threeTreeTypeStats.length"
        class="three-tree-type-panel__message"
      >
        {{
          mapUiText(
            "No tree type data found.",
            "لم يتم العثور على بيانات لأنواع الأشجار.",
            "No tree type data found.",
          )
        }}
      </div>
      <div v-else class="three-tree-type-panel__list">
        <div
          v-for="tree in threeTreeTypeStats"
          :key="tree.name"
          class="three-tree-type-panel__row"
        >
          <span
            class="three-tree-type-panel__icon"
            :class="`is-${tree.archetype}`"
            :style="{ '--tree-icon-color': tree.color }"
            aria-hidden="true"
          >
            <span class="three-tree-type-panel__canopy"></span>
            <span class="three-tree-type-panel__trunk"></span>
          </span>
          <span class="three-tree-type-panel__name" :title="tree.name">
            {{ tree.name }}
          </span>
          <span class="three-tree-type-panel__count">{{ tree.count }}</span>
        </div>
      </div>
    </section>

    <button
      v-if="isStandaloneThreeMode"
      type="button"
      class="three-layer-filters-curtain"
      :class="{ 'is-curtain-up': threeLayerFiltersHidden }"
      :aria-expanded="threeLayerFiltersHidden ? 'false' : 'true'"
      :aria-label="
        threeLayerFiltersHidden
          ? mapUiText('Show Layer Filters', 'إظهار مرشحات الطبقات', 'Show Layer Filters')
          : mapUiText('Hide Layer Filters', 'إخفاء مرشحات الطبقات', 'Hide Layer Filters')
      "
      :title="
        threeLayerFiltersHidden
          ? mapUiText('Pull down to show Layer Filters', 'اسحب لأسفل لإظهار مرشحات الطبقات', 'Pull down to show Layer Filters')
          : mapUiText('Pull up to hide Layer Filters', 'اسحب لأعلى لإخفاء مرشحات الطبقات', 'Pull up to hide Layer Filters')
      "
      @click="threeLayerFiltersHidden = !threeLayerFiltersHidden"
    >
      <span class="three-layer-filters-curtain__folds" aria-hidden="true"></span>
      <span class="three-layer-filters-curtain__pull" aria-hidden="true">
        {{ threeLayerFiltersHidden ? "▼" : "▲" }}
      </span>
    </button>

    <section
      v-if="isStandaloneThreeMode"
      class="three-layer-filters"
      :class="{
        'is-arabic': isArabicUI(),
        'is-curtain-hidden': threeLayerFiltersHidden,
      }"
      :dir="isArabicUI() ? 'rtl' : 'ltr'"
      :aria-label="
        mapUiText('Layer Filters', 'مرشحات الطبقات', 'Layer Filters')
      "
    >
      <div class="three-layer-filters__header">
        <h3 class="three-layer-filters__title">
          {{ mapUiText("Layer Filters", "مرشحات الطبقات", "Layer Filters") }}
        </h3>

        <span
          class="three-layer-filters__visibility-column-title"
          :title="mapUiText('Show layer', 'إظهار الطبقة', 'Show layer')"
          :aria-label="mapUiText('Show layer', 'إظهار الطبقة', 'Show layer')"
        >
          <svg viewBox="0 0 24 24" aria-hidden="true" focusable="false">
            <path
              d="M2.8 12s3.3-5.2 9.2-5.2S21.2 12 21.2 12 17.9 17.2 12 17.2 2.8 12 2.8 12Z"
            />
            <circle cx="12" cy="12" r="2.5" />
          </svg>
        </span>

        <span
          class="three-layer-filters__label-column-title"
          :title="mapUiText('Labels', 'التسميات', 'Labels')"
          :aria-label="mapUiText('Labels', 'التسميات', 'Labels')"
        >
          <svg viewBox="0 0 24 24" aria-hidden="true" focusable="false">
            <path d="M4 5h8.2L20 12.8 12.8 20 5 12.2V5Z" />
            <circle cx="9" cy="9" r="1.35" />
          </svg>
        </span>
      </div>

      <div
        v-for="layer in standaloneThreeLayerOptions"
        :key="layer.key"
        class="three-layer-filter-block"
      >
        <div
          class="three-layer-filter-row"
          :class="{
            'is-disabled':
              (!layer.available && !layer.locked) ||
              (threeParkLoading && !layer.locked),
            'is-locked': layer.locked,
            'is-loading': threeLayerLoadingKey === layer.key,
            'is-partial': layer.partiallyActive,
          }"
        >
          <button
            v-if="layer.expandable"
            type="button"
            class="three-layer-filter-row__arrow"
            :class="{ 'is-expanded': layer.expanded }"
            :aria-expanded="layer.expanded ? 'true' : 'false'"
            :aria-label="
              layer.expanded
                ? mapUiText(
                    `Collapse ${layer.label}`,
                    `طي ${layer.label}`,
                    `Collapse ${layer.label}`,
                  )
                : mapUiText(
                    `Expand ${layer.label}`,
                    `توسيع ${layer.label}`,
                    `Expand ${layer.label}`,
                  )
            "
            :disabled="!layer.available || threeParkLoading"
            @click.stop="toggleStandaloneThreeLayerExpanded(layer.key)"
          >
            <span aria-hidden="true">›</span>
          </button>
          <span
            v-else
            class="three-layer-filter-row__arrow-placeholder"
            aria-hidden="true"
          ></span>

          <label class="three-layer-filter-row__content">
            <span class="three-layer-filter-row__label">
              {{ layer.label }}
            </span>

            <input
              class="three-layer-filter-row__checkbox"
              type="checkbox"
              :checked="layer.active"
              :indeterminate.prop="layer.partiallyActive"
              :disabled="
                (!layer.available && !layer.locked) ||
                (threeParkLoading && !layer.locked) ||
                threeLayerLoadingKey === layer.key
              "
              :aria-checked="layer.partiallyActive ? 'mixed' : undefined"
              :aria-readonly="layer.locked ? 'true' : 'false'"
              :title="
                layer.disabledReason ||
                (layer.locked
                  ? mapUiText(
                      'Terrain is always visible',
                      'طبقة التضاريس ظاهرة دائماً',
                      'Terrain is always visible',
                    )
                  : layer.label)
              "
              @click="handleStandaloneThreeLayerCheckboxClick(layer, $event)"
              @change="handleStandaloneThreeLayerCheckboxChange(layer, $event)"
            />
          </label>

          <label
            v-if="layer.key === 'structures'"
            class="three-layer-filter-row__label-toggle"
            :title="
              mapUiText(
                'Show labels for all structure categories',
                'إظهار تسميات جميع فئات المنشآت',
                'Show labels for all structure categories',
              )
            "
          >
            <input
              class="three-layer-filter-row__checkbox three-layer-filter-row__label-checkbox"
              type="checkbox"
              :checked="isStandaloneStructureLabelParentActive()"
              :indeterminate.prop="
                isStandaloneStructureLabelParentPartiallyActive()
              "
              :disabled="
                !layer.available ||
                !layer.active ||
                threeParkLoading ||
                threeLayerLoadingKey === layer.key
              "
              :aria-label="
                mapUiText(
                  'Structure labels',
                  'تسميات المنشآت',
                  'Structure labels',
                )
              "
              @change="
                handleStandaloneStructureLabelParentCheckboxChange($event)
              "
            />
          </label>
          <span
            v-else
            class="three-layer-filter-row__label-column-placeholder"
            aria-hidden="true"
          ></span>
        </div>

        <div
          v-if="layer.expandable && layer.expanded"
          class="three-layer-filter-children"
        >
          <div
            v-for="child in layer.children"
            :key="`${layer.key}-${child.key}`"
            class="three-layer-filter-child-row"
          >
            <span class="three-layer-filter-child-row__label">
              {{ child.label }}
            </span>
            <input
              class="three-layer-filter-row__checkbox"
              type="checkbox"
              :checked="child.active"
              :disabled="threeParkLoading || threeLayerLoadingKey === layer.key"
              :title="child.label"
              :aria-label="child.label"
              @change="
                handleStandaloneThreeSubLayerCheckboxChange(
                  layer.key,
                  child,
                  $event,
                )
              "
            />
            <input
              v-if="layer.key === 'structures'"
              class="three-layer-filter-row__checkbox three-layer-filter-row__label-checkbox"
              type="checkbox"
              :checked="isStandaloneStructureSubLayerLabelActive(child.key)"
              :disabled="
                !child.active ||
                !threeLayerVisibility.structures ||
                threeParkLoading ||
                threeLayerLoadingKey === layer.key
              "
              :title="
                mapUiText(
                  `Show labels for ${child.label}`,
                  `إظهار تسميات ${child.label}`,
                  `Show labels for ${child.label}`,
                )
              "
              :aria-label="
                mapUiText(
                  `Show labels for ${child.label}`,
                  `إظهار تسميات ${child.label}`,
                  `Show labels for ${child.label}`,
                )
              "
              @change="
                handleStandaloneStructureSubLayerLabelCheckboxChange(
                  child,
                  $event,
                )
              "
            />
            <input
              v-else-if="layer.key === 'vegetation' && child.key === 'trees'"
              class="three-layer-filter-row__checkbox three-layer-filter-row__label-checkbox"
              type="checkbox"
              :checked="threeToolState.treeTypes"
              :disabled="
                !layer.available ||
                threeParkLoading ||
                threeLayerLoadingKey === layer.key
              "
              :title="
                mapUiText(
                  'Tree Type',
                  'أنواع الأشجار',
                  'Tree Type',
                )
              "
              :aria-label="
                mapUiText(
                  'Tree Type',
                  'أنواع الأشجار',
                  'Tree Type',
                )
              "
              @change="handleStandaloneTreeTypeCheckboxChange($event)"
            />
            <span
              v-else
              class="three-layer-filter-row__label-column-placeholder"
              aria-hidden="true"
            ></span>
          </div>
        </div>
      </div>

      <p v-if="threeLayerError" class="three-layer-filters__error" role="alert">
        {{ threeLayerError }}
      </p>
    </section>

    <div
      v-if="shouldShowMainMapFilter"
      class="main-map-layer-switch"
      :class="{ 'is-dark-mode': theme === 'dark', 'is-arabic': isArabicUI() }"
      :dir="isArabicUI() ? 'rtl' : 'ltr'"
    >
      <label class="main-map-layer-switch__label" for="main-map-filter">
        {{ mapUiText("Map Filter", "فلترة الخريطة", "Map Filter") }}
      </label>
      <select
        id="main-map-filter"
        class="main-map-layer-switch__select"
        :value="mainMapView"
        :aria-label="mapUiText('Map Filter', 'فلترة الخريطة', 'Map Filter')"
        @change="setMainMapView($event.target.value)"
      >
        <option
          v-for="filter in mainMapFilters"
          :key="filter.value"
          :value="filter.value"
        >
          {{ mapUiLabel(filter) }}
        </option>
      </select>
    </div>

    <div
      ref="mapNavigationControls"
      class="map-navigation-controls"
      :class="{ 'is-dark-mode': theme === 'dark' }"
      :aria-label="
        mapUiText(
          'Map navigation controls',
          'أزرار تحكم الخريطة',
          'Map navigation controls',
        )
      "
    >
      <button
        v-if="!isStandaloneThreeMode"
        type="button"
        class="map-navigation-button map-location-button"
        :class="{
          'is-locating': isLocatingUser,
          'has-location': hasUserLocation,
        }"
        :title="
          isUserLocationViewActive
            ? mapUiText(
                'Return to default view',
                'العودة إلى العرض الافتراضي',
                'Return to default view',
              )
            : mapUiText(
                'Go to my location',
                'الانتقال إلى موقعي',
                'Go to my location',
              )
        "
        :aria-label="
          isUserLocationViewActive
            ? mapUiText(
                'Return to default view',
                'العودة إلى العرض الافتراضي',
                'Return to default view',
              )
            : mapUiText(
                'Go to my location',
                'الانتقال إلى موقعي',
                'Go to my location',
              )
        "
        @click="goToUserLocation"
      >
        <span class="map-location-arrow" aria-hidden="true"></span>
      </button>
      <button
        type="button"
        class="map-navigation-button map-reset-button"
        :title="
          mapUiText(
            'Reset map position',
            'إعادة تعيين موضع الخريطة',
            'Reset map position',
          )
        "
        :aria-label="
          mapUiText(
            'Reset map position',
            'إعادة تعيين موضع الخريطة',
            'Reset map position',
          )
        "
        @click="resetMapPositionFromControl"
      >
        <span class="map-reset-icon" aria-hidden="true">↺</span>
      </button>
      <button
        type="button"
        class="map-navigation-button map-zoom-button"
        :title="mapUiText('Zoom in', 'تكبير', 'Zoom in')"
        :aria-label="mapUiText('Zoom in', 'تكبير', 'Zoom in')"
        @click="zoomMapIn"
      >
        +
      </button>
      <button
        type="button"
        class="map-navigation-button map-zoom-button"
        :title="mapUiText('Zoom out', 'تصغير', 'Zoom out')"
        :aria-label="mapUiText('Zoom out', 'تصغير', 'Zoom out')"
        @click="zoomMapOut"
      >
        −
      </button>
      <button
        type="button"
        class="map-navigation-button map-fullscreen-button"
        :class="{ 'is-active': isMapExpanded }"
        :title="
          isMapExpanded
            ? mapUiText(
                'Exit full map view',
                'الخروج من عرض الخريطة الكامل',
                'Exit full map view',
              )
            : mapUiText('Expand map', 'تكبير الخريطة', 'Expand map')
        "
        :aria-label="
          isMapExpanded
            ? mapUiText(
                'Exit full map view',
                'الخروج من عرض الخريطة الكامل',
                'Exit full map view',
              )
            : mapUiText('Expand map', 'تكبير الخريطة', 'Expand map')
        "
        :aria-pressed="isMapExpanded ? 'true' : 'false'"
        @click="toggleMapExpanded"
      >
        <svg
          class="map-fullscreen-icon"
          viewBox="0 0 24 24"
          aria-hidden="true"
          focusable="false"
        >
          <path
            v-if="!isMapExpanded"
            d="M4 9V4h5M15 4h5v5M20 15v5h-5M9 20H4v-5"
          />
          <path v-else d="M9 4v5H4M20 9h-5V4M15 20v-5h5M4 15h5v5" />
        </svg>
      </button>
    </div>

    <div v-if="locationErrorMessage" class="map-location-message" role="status">
      {{ locationErrorMessage }}
    </div>

    <div
      v-if="tseLegendVisible && !isStandaloneThreeMode"
      class="tse-map-legend tse-map-legend--overlay"
    >
      <div class="tse-map-legend__title">
        {{ mapUiText("Legend", "المفتاح", "Legend") }}
      </div>
      <button
        type="button"
        class="tse-map-legend__row tse-map-legend__row--clickable"
        :class="{ 'is-disabled': !legendStatusFilters.completed }"
        :aria-pressed="legendStatusFilters.completed ? 'true' : 'false'"
        :title="
          legendStatusFilters.completed
            ? mapUiText('Hide Completed', 'إخفاء المكتمل', 'Hide Completed')
            : mapUiText('Show Completed', 'إظهار المكتمل', 'Show Completed')
        "
        @click="toggleLegendStatusFilter('completed')"
      >
        <span
          class="tse-map-legend__line"
          :class="[
            currentLegendMode === 'tse'
              ? 'is-tse-completed'
              : 'is-default-completed',
            { 'is-muted': !legendStatusFilters.completed },
          ]"
        ></span>
        <span>
          {{ mapUiText("Completed", "مكتمل", "Completed") }}
        </span>
      </button>
      <button
        type="button"
        class="tse-map-legend__row tse-map-legend__row--clickable"
        :class="{ 'is-disabled': !legendStatusFilters.progress }"
        :aria-pressed="legendStatusFilters.progress ? 'true' : 'false'"
        :title="
          legendStatusFilters.progress
            ? mapUiText(
                'Hide In Progress',
                'إخفاء قيد التنفيذ',
                'Hide In Progress',
              )
            : mapUiText(
                'Show In Progress',
                'إظهار قيد التنفيذ',
                'Show In Progress',
              )
        "
        @click="toggleLegendStatusFilter('progress')"
      >
        <span
          class="tse-map-legend__line"
          :class="[
            currentLegendMode === 'tse'
              ? 'is-tse-progress'
              : 'is-default-progress',
            { 'is-muted': !legendStatusFilters.progress },
          ]"
        ></span>
        <span>
          {{ mapUiText("In Progress", "قيد التنفيذ", "In Progress") }}
        </span>
      </button>
      <div v-if="currentLegendMode === 'tse'" class="tse-map-legend__row">
        <span class="tse-map-legend__triangle"></span>
        <span>
          {{ mapUiText("Facilities", "مرافق", "Facilities") }}
        </span>
      </div>
    </div>
  </div>
</template>

<script>
import tseGeometryRows from "../data/tse.json";
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import { createThreeCameraControls } from "../utils/threeCameraControls.js";

// Insert your Mapbox Default Public Token here, or put it in your .env file as:
// VUE_APP_MAPBOX_ACCESS_TOKEN=pk.your_default_public_token_here
const MAPBOX_ACCESS_TOKEN =
  (typeof process !== "undefined" &&
    process.env &&
    process.env.VUE_APP_MAPBOX_ACCESS_TOKEN) ||
  "pk.eyJ1IjoiYmFuZGVyc2FsZWg5MyIsImEiOiJjbXJhajM0ZjUxanFmMzBzOXg3dDZhenkxIn0.-M4BmlndCBFaFpTyQgIDnQ";

const MAPBOX_GL_VERSION = "v3.25.0";
const MAPBOX_GL_JS_URL = `https://api.mapbox.com/mapbox-gl-js/${MAPBOX_GL_VERSION}/mapbox-gl.js`;
const MAPBOX_GL_CSS_URL = `https://api.mapbox.com/mapbox-gl-js/${MAPBOX_GL_VERSION}/mapbox-gl.css`;
const MAPBOX_RTL_TEXT_PLUGIN_URL =
  "https://api.mapbox.com/mapbox-gl-js/plugins/mapbox-gl-rtl-text/v0.3.0/mapbox-gl-rtl-text.js";
const MAPBOX_LANGUAGE_ARABIC = "ar";
const MAPBOX_LANGUAGE_ENGLISH = "en";
const MAPBOX_STYLE_LIGHT = "mapbox://styles/mapbox/streets-v12";
const MAPBOX_STYLE_DARK = "mapbox://styles/mapbox/dark-v11";
const MAPBOX_STYLE_SATELLITE = "mapbox://styles/mapbox/standard-satellite";
const MAPBOX_STYLE_3D = "mapbox://styles/mapbox/standard-satellite";
const THREE_PARK_BACKGROUND = 0x01070b;
const THREE_PARK_SURFACE_COLOR = 0x010609;
const THREE_PARK_WIREFRAME_COLOR = 0x39e9ff;
const THREE_PARK_BOUNDARY_COLOR = 0xa5fbff;
const THREE_PARK_SIDE_COLOR = 0x020b0f;
const THREE_PARK_BASE_COLOR = 0x010406;
// Concrete PATHS palette. The source colours are intentionally a little
// darker than the ArcGIS swatches because ACES tone mapping brightens them on
// screen. This preserves the same visual hierarchy in the 3D scene:
// Roads/Car Parking (dark) < Sidewalk (mid) < Pedestrian Space (bright).
const THREE_PATH_ROAD_PARKING_COLOR = 0x74797b;
const THREE_PATH_ROAD_PARKING_EMISSIVE = 0x171b1c;
const THREE_PATH_ROAD_PARKING_LINE_COLOR = 0xaab0b2;
const THREE_PATH_SIDEWALK_COLOR = 0xa5aaac;
const THREE_PATH_SIDEWALK_EMISSIVE = 0x2a2f30;
const THREE_PATH_SIDEWALK_LINE_COLOR = 0xd0d5d6;
const THREE_PATH_PEDESTRIAN_COLOR = 0xd7dbdc;
const THREE_PATH_PEDESTRIAN_EMISSIVE = 0x3a3f40;
const THREE_PATH_PEDESTRIAN_LINE_COLOR = 0xf0f3f4;
// Paths are ground surfaces, not raised decks. Keep only a tiny clearance to
// avoid z-fighting, then densely drape their triangles over Terrain (after).
const THREE_PATH_TERRAIN_CLEARANCE_METERS = 0.045;
// Sidewalks intentionally sit slightly below every other PATHS sub-layer.
// The remaining clearance keeps them safely above Terrain (after), while
// overlapping pedestrian paths, roads, parking, ramps, etc. cover them.
const THREE_PATH_SIDEWALK_STACK_DROP_METERS = 0.018;
const THREE_PATH_DRAPE_SEGMENT_METERS = 4.0;
const THREE_PATH_DRAPE_MAX_SUBDIVISIONS = 2;

// GroundCover is grass/planting at ground level. Keep it almost flush with
// Terrain (after) and drape it over elevation changes instead of face-lifting
// the whole polygon above the highest terrain sample.
const THREE_GROUNDCOVER_TERRAIN_CLEARANCE_METERS = 0.025;
const THREE_GROUNDCOVER_DRAPE_SEGMENT_METERS = 5.0;
const THREE_GROUNDCOVER_DRAPE_MAX_SUBDIVISIONS = 10;

// Path outlines can be sampled more coarsely than the terrain-draped fill.
// The shader grid already communicates elevation, so this saves CPU/memory
// without changing the visible path surface quality.
const THREE_PATH_OUTLINE_SEGMENT_METERS = 6.0;
// Regular square grid drawn directly in the PATHS surface shader. Unlike a
// triangle wireframe, this reads as a survey/elevation grid and visibly bends
// with the draped path surface when terrain elevation changes.
const THREE_PATH_GRID_CELL_METERS = 2.0;
const THREE_PATH_GRID_OPACITY = 0.75;
// SEALED GROUND should retain a clearly readable survey/elevation grid over
// its repeating texture so slopes remain legible at close camera angles.
const THREE_SEALED_GROUND_GRID_CELL_METERS = 2.5;
const THREE_SEALED_GROUND_GRID_OPACITY = 0.68;
// Physical structures begin slightly above ground/path surfaces so their bases
// remain visible and read as sitting on the paving rather than under it.
const THREE_STRUCTURE_SURFACE_CLEARANCE_METERS = 0.09;
// Covered pergolas use the warm olive-tan canopy colour shown in the design
// reference instead of inheriting the purple Structures palette.
const THREE_PARK_PERGOLA_SOLID_COLOR = 0xb6ad69;
const THREE_PARK_PERGOLA_EMISSIVE_COLOR = 0x4d4726;
const THREE_PARK_PERGOLA_LINE_COLOR = 0xeee6af;
const THREE_PARK_MAX_TRIANGLES = 24000;
const THREE_PARK_MIN_SUBDIVISIONS = 5;
const THREE_PARK_MAX_SUBDIVISIONS = 24;
const THREE_PARK_IDLE_TIMEOUT_MS = 9000;
// The floating-island wall starts at the sampled terrain edge and drops to a
// shared lower plane. Its top therefore follows the real perimeter elevation,
// while the adaptive depth keeps the island visible without recreating a tall
// cyan box.
const THREE_PARK_MIN_ISLAND_WALL_DEPTH_METERS = 28;
const THREE_PARK_MAX_ISLAND_WALL_DEPTH_METERS = 120;
const THREE_PARK_ISLAND_WALL_DEPTH_RATIO = 0.045;
const THREE_PARK_MIN_BOUNDARY_SAMPLE_SPACING_METERS = 7;
const THREE_PARK_MAX_BOUNDARY_SAMPLE_SPACING_METERS = 32;
const THREE_PARK_BOUNDARY_SAMPLE_DIVISIONS = 120;
const THREE_PARK_STATIC_TEXTURE_MAX_DIMENSION = 640;
const THREE_PARK_STATIC_TEXTURE_MIN_DIMENSION = 224;
const THREE_PARK_STATIC_TEXTURE_TIMEOUT_MS = 12000;
// Local repeating material used by the optional SEALED GROUND terrain
// texture. Put the supplied image at public/images/sealed_ground.png.
const THREE_PARK_COMPACTED_GROUND_TEXTURE_URL = "/images/sealed_ground.png";
const THREE_PARK_COMPACTED_GROUND_TILE_METERS = 18;
const THREE_PARK_MIN_VERTICAL_EXAGGERATION = 1;
const THREE_PARK_MAX_VERTICAL_EXAGGERATION = 9;
// Target small 4–8 m holographic cells. This is close to the concept art,
// where a typical tree occupies about one surface-grid cell.
const THREE_PARK_GRID_DIVISIONS = 260;
const THREE_PARK_MIN_GRID_SPACING = 4;
const THREE_PARK_MAX_GRID_SPACING = 8;
// Smaller wall cells make the floating-island cut face look closer to the
// concept render instead of drawing large rectangular panels.
const THREE_PARK_WALL_GRID_HORIZONTAL_DIVISIONS = 8;
const THREE_PARK_WALL_GRID_VERTICAL_SAMPLE_STEP = 1;
const THREE_PARK_BLANK_STYLE = {
  version: 8,
  sources: {},
  layers: [
    {
      id: "three-park-background",
      type: "background",
      paint: {
        "background-color": "#061218",
      },
    },
  ],
};
// Satellite uses Mapbox's complete 3D environment. Terrain is deliberately
// enhanced enough to reveal Riyadh landforms while keeping the satellite image,
// labels, atmosphere and extruded buildings visible together.
const MAPBOX_TERRAIN_SOURCE_ID = "green-riyadh-mapbox-dem";
// Do not depend on the current basemap style to expose building data.
// Mapbox Standard Satellite can show terrain well but may not expose/render
// the built-in 3D building layer, so add our own building vector source.
const MAPBOX_3D_BUILDINGS_SOURCE_ID = "green-riyadh-mapbox-buildings";
const MAPBOX_3D_BUILDINGS_LAYER_ID = "green-riyadh-mapbox-3d-buildings";
const MAPBOX_3D_BUILDINGS_SAMPLING_LAYER_ID =
  "green-riyadh-mapbox-3d-buildings-sampling";
const MAPBOX_3D_BUILDINGS_MIN_ZOOM = 12;
const MAPBOX_3D_BUILDINGS_CAMERA_ZOOM = 17.1;
const MAPBOX_3D_BUILDING_HEIGHT_EXAGGERATION = 2.0;
const MAPBOX_3D_BUILDING_MIN_HEIGHT = 3;
const THREE_PARK_BUILDING_QUERY_MAX_ZOOM = 16.8;
const THREE_PARK_BUILDING_TILE_SETTLE_MS = 160;
const THREE_PARK_BUILDING_MIN_INSIDE_RATIO = 0.98;
const THREE_PARK_BUILDING_QUERY_GRID_MAX_COLUMNS = 5;
const THREE_PARK_BUILDING_QUERY_GRID_MAX_ROWS = 5;
const THREE_PARK_BUILDING_QUERY_VIEWPORT_COVERAGE_RATIO = 0.62;
const THREE_PARK_BUILDING_MAX_FOOTPRINT_DIMENSION_METERS = 260;
const THREE_PARK_BUILDING_MAX_FOOTPRINT_AREA_SQUARE_METERS = 30000;
const MAPBOX_BUILDING_TILEQUERY_TILESET_ID = "mapbox.mapbox-streets-v8";
const THREE_PARK_BUILDING_TILEQUERY_RADIUS_METERS = 95;
const THREE_PARK_BUILDING_TILEQUERY_LIMIT = 100;
const THREE_PARK_BUILDING_TILEQUERY_MAX_POINTS = 0;
const THREE_PARK_BUILDING_TILEQUERY_TIMEOUT_MS = 900;
const THREE_PARK_BUILDING_TILEQUERY_BATCH_SIZE = 5;
const MAPBOX_BUILDING_QUERY_SOURCE_LAYERS = [
  "building",
  "structure",
  "landmark",
];
const MAPBOX_BUILDING_POINT_FOOTPRINT_METERS = 18;
const MAPBOX_3D_TERRAIN_EXAGGERATION = 1;
// Slow down mouse-wheel / trackpad zoom so users can make small, accurate
// zoom adjustments without relying only on the + / − buttons.
const MAPBOX_MOUSE_WHEEL_ZOOM_RATE = 1 / 1300;
const MAPBOX_TRACKPAD_ZOOM_RATE = 1 / 320;
const THREE_PARK_MOUSE_WHEEL_ZOOM_SPEED = 0.85;
const MAPBOX_3D_TERRAIN_PITCH = 78;
const MAPBOX_3D_TERRAIN_BEARING = -28;
const MAPBOX_NEUTRAL_ROAD_COLOR_LIGHT = "#cfd6d2";
const MAPBOX_NEUTRAL_ROAD_OUTLINE_LIGHT = "#f4f5f3";
const MAPBOX_NEUTRAL_ROAD_COLOR_DARK = "#4b555a";
const MAPBOX_NEUTRAL_ROAD_OUTLINE_DARK = "#242b2f";
const MAPBOX_NEUTRAL_ROAD_LABEL_LIGHT = "#6f787c";
const MAPBOX_NEUTRAL_ROAD_LABEL_DARK = "#b8c0c4";
const MAPBOX_NEUTRAL_ROAD_OPACITY_LIGHT = 0.48;
const MAPBOX_NEUTRAL_ROAD_OPACITY_DARK = 0.42;
const MAPBOX_NEUTRAL_ROAD_OUTLINE_OPACITY_LIGHT = 0.32;
const MAPBOX_NEUTRAL_ROAD_OUTLINE_OPACITY_DARK = 0.28;
const MAPBOX_NEUTRAL_ROAD_SHIELD_OPACITY_LIGHT = 0.22;
const MAPBOX_NEUTRAL_ROAD_SHIELD_OPACITY_DARK = 0.26;
const MAPBOX_NEUTRAL_ROAD_LABEL_OPACITY_LIGHT = 0.55;
const MAPBOX_NEUTRAL_ROAD_LABEL_OPACITY_DARK = 0.6;
const MAPBOX_NEUTRAL_PARK_FILL_LIGHT = "#e0ddd4";
const MAPBOX_NEUTRAL_PARK_OUTLINE_LIGHT = "#d0cec6";
const MAPBOX_NEUTRAL_PARK_FILL_DARK = "#30383b";
const MAPBOX_NEUTRAL_PARK_OUTLINE_DARK = "#465155";
const MAPBOX_NEUTRAL_PARK_FILL_OPACITY_LIGHT = 0.5;
const MAPBOX_NEUTRAL_PARK_FILL_OPACITY_DARK = 0.42;
const MAPBOX_NEUTRAL_PARK_LINE_OPACITY_LIGHT = 0.35;
const MAPBOX_NEUTRAL_PARK_LINE_OPACITY_DARK = 0.38;

const PROJECT_SOURCE_ID = "green-riyadh-projects";
const DISTRICT_SOURCE_ID = "green-riyadh-districts";
const PROJECT_FILL_LAYER_ID = "green-riyadh-project-fills";
const PROJECT_LINE_LAYER_ID = "green-riyadh-project-lines";
const PROJECT_FACILITY_LAYER_ID = "green-riyadh-project-facilities";
const DISTRICT_FILL_LAYER_ID = "green-riyadh-district-fills";
const DISTRICT_LINE_LAYER_ID = "green-riyadh-district-lines";
const MAPBOX_DEFAULT_MAX_BOUNDS = [
  [46.0, 24.02],
  [47.4, 26.02],
];
const QADISIYAH_3_PROJECT_ID = "22";
const QADISIYAH_3_PROJECT_CODE = "GP.PK.CP.140";
const QADISIYAH_3_PROJECT_NAME = "qadisiyah 3 park";
const QADISIYAH_3_CONCEPT_IMAGE_SOURCE_ID =
  "green-riyadh-qadisiyah-3-concept-image";
const QADISIYAH_3_CONCEPT_IMAGE_LAYER_ID =
  "green-riyadh-qadisiyah-3-concept-image-layer";
const QADISIYAH_3_CONCEPT_MASK_SOURCE_ID =
  "green-riyadh-qadisiyah-3-concept-mask";
const QADISIYAH_3_CONCEPT_MASK_LAYER_ID =
  "green-riyadh-qadisiyah-3-concept-mask-layer";
const QADISIYAH_3_CONCEPT_FADE_LAYER_ID =
  "green-riyadh-qadisiyah-3-concept-fade-layer";
const QADISIYAH_3_CONCEPT_MASK_METERS = 240;
const QADISIYAH_3_CONCEPT_CAMERA_METERS = 390;
const QADISIYAH_3_CONCEPT_IMAGE_OPACITY = 0.72;
// All Architecture concept sheets were prepared with the same park footprint
// alignment, so use one shared crop for every concept tab. This removes the
// remaining thin white strip at the top and keeps every overlay aligned to the
// same map geometry corners.
const QADISIYAH_3_CONCEPT_SHARED_CROP = {
  left: 0.0345,
  top: 0.1925,
  right: 0.324,
  bottom: 0.4375,
};
// Mask the cropped concept sheet to the park design footprint before sending
// it to Mapbox, so pixels outside the intended project space stay transparent
// instead of being rendered underneath the black silhouette. Coordinates are
// normalized within the cropped image canvas.
const QADISIYAH_3_CONCEPT_IMAGE_MASK_POINTS = [
  [0.018, 0.105],
  [0.962, 0.232],
  [0.978, 0.805],
  [0.012, 0.742],
];
// Crop the Architecture sheets before sending them to Mapbox.
// This shared crop removes page titles, legends, white margins, and extra
// sheet area so only the aligned park design footprint is stretched over the
// project geometry. Values are percentages of image size.
const QADISIYAH_3_CONCEPT_DEFAULT_CROP = QADISIYAH_3_CONCEPT_SHARED_CROP;
// The concept drawings are plan-view rasters, not real GIS layers. These four
// coordinates drape the plan over Qadisiyah 3 Park using the project polygon's
// long north/south edges so Architecture's sheets align with the same site area.
const QADISIYAH_3_CONCEPT_IMAGE_COORDINATES = [
  [46.819826, 24.831797],
  [46.830011, 24.833977],
  [46.830673, 24.832667],
  [46.821367, 24.828749],
];
const QADISIYAH_3_CONCEPT_LAYERS = [
  {
    id: "information",
    labelKey: "Information",
    labelEn: "Information",
    labelAr: "المعلومات",
    fallbackImage: "/projects_images/AlQadisiyah(3)Park/mapinfo-1.png",
    imageIndex: 0,
    // Geometry-fit crop for the main Qadisiyah 3 park polygon.
    // This intentionally excludes the page title, bottom legend, and the
    // detached eastern concept extension so the four visible sheet corners
    // align as closely as possible with the four map geometry corners.
    // 1536x1024 sheet crop approx: x 53-1038, y 184-576.
    crop: QADISIYAH_3_CONCEPT_SHARED_CROP,
  },
  {
    id: "landscape",
    labelKey: "Landscape",
    labelEn: "Landscape",
    labelAr: "التصميم",
    fallbackImage: "/projects_images/AlQadisiyah(3)Park/mapinfo-2.png",
    imageIndex: 1,
    // Geometry-fit crop for the main Qadisiyah 3 park polygon.
    // Uses the same west/east limits as Information so the main trapezoid
    // aligns to the project geometry rather than the larger sheet.
    // 1536x1024 sheet crop approx: x 53-1038, y 182-575.
    crop: QADISIYAH_3_CONCEPT_SHARED_CROP,
  },
  {
    id: "buildings",
    labelKey: "Buildings",
    labelEn: "Buildings",
    labelAr: "المباني",
    fallbackImage: "/projects_images/AlQadisiyah(3)Park/mapinfo-3.png",
    imageIndex: 2,
    // 1920x1280 sheet crop: removes the title and bottom legend.
    crop: QADISIYAH_3_CONCEPT_SHARED_CROP,
  },
  {
    id: "signage",
    labelKey: "Signage",
    labelEn: "Signage",
    labelAr: "اللوحات",
    fallbackImage: "/projects_images/AlQadisiyah(3)Park/mapinfo-4.png",
    imageIndex: 3,
    crop: QADISIYAH_3_CONCEPT_SHARED_CROP,
  },
  {
    id: "paths",
    labelKey: "Paths",
    labelEn: "Paths",
    labelAr: "المسارات",
    fallbackImage: "/projects_images/AlQadisiyah(3)Park/mapinfo-5.png",
    imageIndex: 4,
    crop: QADISIYAH_3_CONCEPT_SHARED_CROP,
  },
  {
    id: "lighting",
    labelKey: "Lighting",
    labelEn: "Lighting",
    labelAr: "الإضاءة",
    fallbackImage: "/projects_images/AlQadisiyah(3)Park/mapinfo-6.png",
    imageIndex: 5,
    crop: QADISIYAH_3_CONCEPT_SHARED_CROP,
  },
  {
    id: "parking",
    labelKey: "Parking",
    labelEn: "Parking",
    labelAr: "المواقف",
    fallbackImage: "/projects_images/AlQadisiyah(3)Park/mapinfo-7.png",
    imageIndex: 6,
    crop: QADISIYAH_3_CONCEPT_SHARED_CROP,
  },
  {
    id: "trees",
    labelKey: "Trees",
    labelEn: "Trees",
    labelAr: "الأشجار",
    fallbackImage: "/projects_images/AlQadisiyah(3)Park/mapinfo-8.png",
    imageIndex: 7,
    crop: QADISIYAH_3_CONCEPT_SHARED_CROP,
  },
];
const EMPTY_FEATURE_COLLECTION = {
  type: "FeatureCollection",
  features: [],
};

let mapboxLoadPromise = null;
let mapboxRtlTextPluginRequested = false;

function isMapboxTokenConfigured(token) {
  return (
    typeof token === "string" &&
    token.trim().startsWith("pk.") &&
    token !== "PASTE_YOUR_MAPBOX_DEFAULT_PUBLIC_TOKEN_HERE"
  );
}

function loadMapboxCss() {
  if (document.querySelector(`link[href="${MAPBOX_GL_CSS_URL}"]`)) {
    return Promise.resolve();
  }

  return new Promise((resolve, reject) => {
    const link = document.createElement("link");
    link.rel = "stylesheet";
    link.href = MAPBOX_GL_CSS_URL;
    link.onload = resolve;
    link.onerror = () => reject(new Error("Mapbox GL CSS failed to load."));
    document.head.appendChild(link);
  });
}

function loadMapboxScript() {
  if (window.mapboxgl && typeof window.mapboxgl.Map === "function") {
    return Promise.resolve(window.mapboxgl);
  }

  const existingScript = document.querySelector(
    `script[src="${MAPBOX_GL_JS_URL}"]`,
  );

  if (existingScript) {
    return new Promise((resolve, reject) => {
      existingScript.addEventListener("load", () => resolve(window.mapboxgl), {
        once: true,
      });
      existingScript.addEventListener(
        "error",
        () => reject(new Error("Mapbox GL JS failed to load.")),
        { once: true },
      );
    });
  }

  return new Promise((resolve, reject) => {
    const script = document.createElement("script");
    script.src = MAPBOX_GL_JS_URL;
    script.async = true;
    script.defer = true;
    script.onload = () => resolve(window.mapboxgl);
    script.onerror = () => reject(new Error("Mapbox GL JS failed to load."));
    document.head.appendChild(script);
  });
}

function ensureMapboxRtlTextPlugin(mapboxgl) {
  if (!mapboxgl || typeof mapboxgl.setRTLTextPlugin !== "function") return;

  try {
    const status =
      typeof mapboxgl.getRTLTextPluginStatus === "function"
        ? mapboxgl.getRTLTextPluginStatus()
        : null;

    if (
      status === "loaded" ||
      status === "loading" ||
      mapboxRtlTextPluginRequested
    ) {
      return;
    }

    mapboxRtlTextPluginRequested = true;
    mapboxgl.setRTLTextPlugin(MAPBOX_RTL_TEXT_PLUGIN_URL, null, true);
  } catch (error) {
    // If another map already loaded the plugin, Mapbox can throw. Arabic labels
    // may still work if the plugin was already registered, so do not block map load.
    mapboxRtlTextPluginRequested = true;
  }
}

async function loadMapboxGL() {
  if (!mapboxLoadPromise) {
    mapboxLoadPromise = Promise.all([loadMapboxCss(), loadMapboxScript()]).then(
      ([, mapboxgl]) => mapboxgl || window.mapboxgl,
    );
  }

  return mapboxLoadPromise;
}

export default {
  props: {
    latitude: {
      type: Number,
      default() {
        return this.$root.$data.center[0];
      },
    },
    longitude: {
      type: Number,
      default() {
        return this.$root.$data.center[1];
      },
    },
    zoom: {
      type: Number,
      default() {
        return 10;
      },
    },
  },
  data() {
    return {
      theme: "light",
      isMapExpanded: false,
      mapExpandedLeft: 0,
      threeLayerFiltersHidden: false,
      mapboxgl: null,
      mapReadyPromise: null,
      mapReadyResolve: null,
      allProjectFeatures: [],
      renderedProjectFeatures: [],
      districtFeatures: [],
      tseGeometryByUid: null,
      mapPopup: null,
      markers: [],
      geoJson: [],
      zoom_changed_listener: null,
      contractMapFilter: "all",
      mainMapView: "all",
      activeBaseMapMode: "street",
      baseMapModes: [
        {
          value: "street",
          labelKey: "Street",
          labelEn: "Street",
          labelAr: "شوارع",
        },
        {
          value: "satellite",
          labelKey: "Satellite",
          labelEn: "Satellite",
          labelAr: "قمر صناعي",
        },
        // Keep the Arabic 3D label short so the style buttons do not collide
        // with the map-filter panel on narrow map columns.
        { value: "3d", labelKey: "3D", labelEn: "3D", labelAr: "مجسم" },
      ],
      mainMapFilters: [
        {
          value: "all",
          category: "all",
          labelKey: "All Projects",
          labelEn: "All Projects",
          labelAr: "كل المشاريع",
        },
        {
          value: "large-parks",
          category: "Large Parks",
          labelKey: "Parks",
          labelEn: "Parks",
          labelAr: "الحدائق",
        },
        {
          value: "neighborhoods",
          category: "Green Neighborhoods",
          labelKey: "Neighborhoods",
          labelEn: "Neighborhoods",
          labelAr: "الأحياء",
        },
        {
          value: "natural-parks",
          category: "Natural Parks",
          labelKey: "Wadis",
          labelEn: "Wadis",
          labelAr: "الأودية",
        },
        {
          value: "main-roads",
          category: "Main Roads",
          labelKey: "Roads",
          labelEn: "Roads",
          labelAr: "الطرق",
        },
        {
          value: "public-complex",
          category: "Public Complexes",
          labelKey: "Complexes",
          labelEn: "Complexes",
          labelAr: "المجمعات",
        },
        {
          value: "nurseries",
          category: "Nurseries",
          labelKey: "Nurseries",
          labelEn: "Nurseries",
          labelAr: "المشاتل",
        },
        {
          value: "tse",
          category: "TSE Network & Water",
          labelKey: "TSE",
          labelEn: "TSE",
          labelAr: "مياه الري",
        },
      ],
      currentMarkerCategory: null,
      currentBackRouter: null,
      tseLegendVisible: false,
      currentLegendMode: "default",
      legendStatusFilters: {
        completed: true,
        progress: true,
      },
      activeSearchProjectIds: null,
      highlightedProjectId: null,
      hoveredFeatureKey: null,
      currentProjectId: null,
      nearProjectId: null,
      nearProjectCenter: null,
      activeViewMode: "all",
      userLocationMarker: null,
      userLocationWatchId: null,
      deviceOrientationHandler: null,
      userHeading: 0,
      hasUserLocation: false,
      isUserLocationViewActive: false,
      isLocatingUser: false,
      locationErrorMessage: "",
      locationMessageTimeout: null,
      mapLayerEventsBound: false,
      projectLayersReady: false,
      mapOverlayResizeHandler: null,
      mapOverlayResizeObserver: null,
      middleMousePanHandlers: null,
      middleMousePanState: null,
      mapboxLanguageObserver: null,
      mapboxLanguageSyncInterval: null,
      mapboxLanguageRootHandlers: [],
      currentMapboxLanguage: null,
      selectedConceptLayerId: "information",
      conceptMapBoundsActive: false,
      conceptImageUrlCache: {},
      threeParkLoading: false,
      threeParkError: "",
      // "before" is the existing Mapbox DEM surface. "after" is reserved
      // for a future proposed-grading/elevation dataset.
      threeTerrainMode: "before",
      threeLayerLoadingKey: "",
      threeLayerError: "",
      // Keep GIS-loading failures separate from terrain failures. A bad/missing
      // GIS file must never prevent the base Terrain (before)/(after) shell
      // from opening in standalone 3D.
      threeProjectGisLoadError: "",
      threeLayerVisibility: {
        terrain: true,
        satelliteGround: false,
        compactedGround: false,
        paths: false,
        water: false,
        structures: false,
        vegetation: false,
        tse: false,
      },
      threeLayerAvailability: {
        terrainAfter: true,
        paths: false,
        water: false,
        structures: false,
        vegetation: false,
        tse: false,
      },
      threeLayerExpanded: {
        paths: false,
        water: false,
        structures: false,
        vegetation: false,
        tse: false,
      },
      threeSubLayerAvailability: {
        paths: [],
        water: [],
        structures: [],
        vegetation: [],
        tse: [],
      },
      threeSubLayerVisibility: {
        paths: {},
        water: {},
        structures: {},
        vegetation: {},
        tse: {},
      },
      threeToolState: {
        structureLabels: false,
        conceptPaths: false,
        treeTypes: false,
      },
      // Per-structure-category label switches shown as the second checkbox
      // column in Layer Filters. Keys match the Structures sub-layer keys.
      threeStructureLabelVisibility: {},
      threeTreeTypeLoading: false,
      threeTreeTypeStats: [],
      threeStructureAnnotations: [],
      threeVisibleStructureLabels: [],
      icons: {
        active: {
          icon: "/images/marker-selected.gif",
        },
        inactive: {
          icon: "/images/marker-unselected-light2.png",
        },
      },
    };
  },
  computed: {
    mapExpandedStyle() {
      return this.isMapExpanded
        ? { "--green-riyadh-expanded-map-left": `${this.mapExpandedLeft}px` }
        : null;
    },
    isProjectDetailsMap() {
      return (
        this.activeViewMode === "project" &&
        Boolean(this.currentProjectId) &&
        this.$route &&
        this.$route.name === "projects"
      );
    },

    isStandaloneThreeMode() {
      return this.activeBaseMapMode === "3d" && this.isProjectDetailsMap;
    },

    shouldShowMainMapFilter() {
      return (
        !this.isStandaloneThreeMode &&
        this.$route &&
        this.$route.name === "Home"
      );
    },

    visibleBaseMapModes() {
      return this.baseMapModes.filter(
        (mode) => mode.value !== "3d" || this.isProjectDetailsMap,
      );
    },

    threeTreeTypeTotal() {
      return (
        Array.isArray(this.threeTreeTypeStats) ? this.threeTreeTypeStats : []
      ).reduce((sum, item) => sum + (Number(item && item.count) || 0), 0);
    },

    standaloneThreeLayerOptions() {
      const createExpandableLayer = (key, label, available) => {
        const children = this.getStandaloneThreeSubLayerOptions(key);
        return {
          key,
          label,
          available: Boolean(available),
          locked: false,
          active: this.isStandaloneThreeParentLayerActive(key),
          partiallyActive:
            this.isStandaloneThreeParentLayerPartiallyActive(key),
          expandable: children.length > 1,
          expanded: Boolean(this.threeLayerExpanded[key]),
          children,
        };
      };

      return [
        {
          key: "terrainBefore",
          label: this.mapUiText(
            "Terrain (before)",
            "التضاريس (قبل)",
            "Terrain (before)",
          ),
          available: true,
          locked: false,
          active:
            Boolean(this.threeLayerVisibility.terrain) &&
            this.threeTerrainMode === "before",
          partiallyActive: false,
          expandable: false,
          expanded: false,
          children: [],
        },
        {
          key: "terrainAfter",
          label: this.mapUiText(
            "Terrain (after)",
            "التضاريس (بعد)",
            "Terrain (after)",
          ),
          available: true,
          locked: false,
          active:
            Boolean(this.threeLayerVisibility.terrain) &&
            this.threeTerrainMode === "after",
          partiallyActive: false,
          expandable: false,
          expanded: false,
          children: [],
          disabledReason: "",
        },
        {
          key: "satelliteGround",
          label: this.mapUiText(
            "Satellite Ground",
            "أرضية القمر الصناعي",
            "Satellite Ground",
          ),
          // Satellite Ground is a texture applied to whichever terrain surface
          // is selected. It cannot be changed while both terrain options are off.
          available: Boolean(this.threeLayerVisibility.terrain),
          locked: false,
          active: Boolean(this.threeLayerVisibility.satelliteGround),
          partiallyActive: false,
          expandable: false,
          expanded: false,
          children: [],
          disabledReason: this.threeLayerVisibility.terrain
            ? ""
            : this.mapUiText(
                "Enable Terrain (before) or Terrain (after) first",
                "فعّل التضاريس (قبل) أو التضاريس (بعد) أولاً",
                "Enable Terrain (before) or Terrain (after) first",
              ),
        },
        {
          key: "compactedGround",
          label: this.mapUiText(
            "SEALED GROUND",
            "أرضية مختومة",
            "SEALED GROUND",
          ),
          // SEALED GROUND uses the same terrain surface and grid as
          // Satellite Ground, but replaces the colour with a local repeating
          // sealed-ground texture from /public/images.
          available: Boolean(this.threeLayerVisibility.terrain),
          locked: false,
          active: Boolean(this.threeLayerVisibility.compactedGround),
          partiallyActive: false,
          expandable: false,
          expanded: false,
          children: [],
          disabledReason: this.threeLayerVisibility.terrain
            ? ""
            : this.mapUiText(
                "Enable Terrain (before) or Terrain (after) first",
                "فعّل التضاريس (قبل) أو التضاريس (بعد) أولاً",
                "Enable Terrain (before) or Terrain (after) first",
              ),
        },
        createExpandableLayer(
          "paths",
          this.mapUiText("Paths", "المسارات", "Paths"),
          this.threeLayerAvailability.paths,
        ),
        createExpandableLayer(
          "water",
          this.mapUiText("Water", "المياه", "Water"),
          this.threeLayerAvailability.water,
        ),
        createExpandableLayer(
          "structures",
          this.mapUiText("Structures", "المنشآت", "Structures"),
          this.threeLayerAvailability.structures,
        ),
        createExpandableLayer(
          "vegetation",
          this.mapUiText("Vegetation", "النباتات", "Vegetation"),
          this.threeLayerAvailability.vegetation,
        ),
        createExpandableLayer(
          "tse",
          this.mapUiText("Utility", "المرافق", "Utility"),
          this.threeLayerAvailability.tse,
        ),
      ];
    },

    standaloneThreeToolOptions() {
      return [
        {
          key: "focus",
          enabled: false,
          label: this.mapUiText(
            "Focus tool (coming soon)",
            "أداة التركيز (قريباً)",
            "Focus tool (coming soon)",
          ),
        },
        {
          key: "layers",
          enabled: Boolean(this.threeLayerAvailability.vegetation),
          label: this.mapUiText("Tree Type", "أنواع الأشجار", "Tree Type"),
        },
        {
          key: "measure",
          enabled: Boolean(this.threeLayerAvailability.paths),
          label: this.mapUiText(
            this.threeToolState.conceptPaths
              ? "Hide concept paths"
              : "Show concept paths",
            this.threeToolState.conceptPaths
              ? "إخفاء المسارات التصميمية"
              : "إظهار المسارات التصميمية",
            this.threeToolState.conceptPaths
              ? "Hide concept paths"
              : "Show concept paths",
          ),
        },
        {
          key: "structureLabels",
          enabled: Boolean(
            this.threeLayerVisibility.structures ||
            (this.threeLayerGroups && this.threeLayerGroups.structures) ||
            this.threeLayerAvailability.structures,
          ),
          label: this.mapUiText(
            this.threeToolState.structureLabels
              ? "Hide structure labels"
              : "Show structure labels",
            this.threeToolState.structureLabels
              ? "إخفاء أسماء عناصر المنشآت"
              : "إظهار أسماء عناصر المنشآت",
            this.threeToolState.structureLabels
              ? "Hide structure labels"
              : "Show structure labels",
          ),
        },
      ];
    },

    showConceptMapControls() {
      return this.isConceptMapMode() && this.isQadisiyah3ProjectActive();
    },

    visibleConceptMapLayers() {
      const qadisiyahItem = this.getQadisiyah3ProjectItem();
      const images = this.getConceptImageList(qadisiyahItem);
      return QADISIYAH_3_CONCEPT_LAYERS.filter((layer) => {
        const imagePath = this.getConceptLayerImagePath(layer, images);
        return Boolean(imagePath);
      });
    },
  },
  watch: {
    $route(to, from) {
      // Full-screen/expanded map mode belongs to a single page only. Never
      // carry it into the next route.
      if (this.isMapExpanded) {
        this.setMapExpanded(false);
      }

      // Leaving ProjectDetails should always return the shared map component
      // to its normal default state before the destination page configures it.
      if (this.isProjectDetailsRoute(from) && !this.isProjectDetailsRoute(to)) {
        this.resetMapAfterProjectDetails();
      }
    },
  },

  async mounted() {
    // Keep complex Three.js objects outside Vue 2's reactive data observer.
    this.threeRenderer = null;
    this.threeScene = null;
    this.threeCamera = null;
    this.threeControls = null;
    this.threeParkGroup = null;
    this.threeParkModelData = null;
    this.threeLayerGroups = {
      terrain: null,
      terrainBefore: null,
      terrainAfter: null,
      paths: null,
      water: null,
      structures: null,
      vegetation: null,
      tse: null,
    };
    this.threeProjectGisCache = new Map();
    this.threeProjectGisData = null;
    this.threeProjectGisUrl = "";
    this.threeProjectGisFallbackLayer = "";
    this.threeVegetationLoadToken = 0;
    this.threeParkAnimationFrame = null;
    this.threeParkResizeObserver = null;
    this.threeCameraControls = null;
    this.threeParkBuildToken = 0;
    this.threeSatelliteGroundLoadToken = 0;
    this.threeSatelliteGroundTexture = null;
    this.threeCompactedGroundLoadToken = 0;
    this.threeCompactedGroundTexture = null;
    this.threeDefaultCameraState = null;
    this.threeLabelOverlayUpdateAt = 0;
    this.threeLabelOverlaySignature = "";
    this.standaloneBuildingFeatureCache = [];
    this.satelliteTerrainRestoreTimers = [];
    this.pendingSatelliteTerrainHardRefresh = false;
    this.lastStandaloneThreeResetMapViewState = null;
    // Preserve center/zoom/pitch/bearing while changing Street, Satellite and
    // the standalone Three.js 3D renderer. These are kept outside Vue 2's
    // reactive observer because they are transient renderer state.
    this.preservedMapViewState = null;
    this.pendingBaseMapCameraState = null;
    this.mapCameraRestoreTimers = [];
    this.isBaseMapStyleSwitching = false;
    this.mapboxRendererSuspended = false;
    this.suspendMapStyleRestoration = false;
    this.mapExpandedKeydownHandler = (event) => {
      if (event.key === "Escape" && this.isMapExpanded) {
        this.setMapExpanded(false);
      }
    };
    window.addEventListener("keydown", this.mapExpandedKeydownHandler);

    this.theme =
      window.localStorage.theme == undefined
        ? "light"
        : window.localStorage.theme;

    this.$root.$on("change-mode", (type) => {
      this.theme = type;
      this.set_map_dark(type);
      this.$nextTick(() => {
        this.syncMapPopupTheme();
      });
    });

    try {
      await this.ensureMapboxLoaded();
      await this.init_map();
      this.startMapboxLanguageSync();
      this.updateMapOverlayControlPositions();
      this.mapOverlayResizeHandler = () => {
        this.updateMapOverlayControlPositions();
        this.syncThreeParkHostBounds();
        this.resizeStandaloneThreeRenderer();
      };
      window.addEventListener("resize", this.mapOverlayResizeHandler);
      if (typeof ResizeObserver !== "undefined" && this.$refs.map) {
        this.mapOverlayResizeObserver = new ResizeObserver(
          this.mapOverlayResizeHandler,
        );
        this.mapOverlayResizeObserver.observe(this.$refs.map);
      }
    } catch (error) {
      console.error("Mapbox map could not be loaded:", error);
      this.locationErrorMessage = this.isArabicUI()
        ? "تعذر تحميل الخريطة. يرجى تحديث الصفحة أو التحقق من اتصال الشبكة."
        : "The map could not be loaded. Please refresh the page or check the network connection.";
    }
  },
  created() {
    this.$root.$on("load-project-map", (item) => {
      this.load_project(item);
    });
    this.$root.$on("load-dist-map", (item, district, action = true) => {
      this.load_near_projects(item, district, action);
    });
    this.$root.$on("set-map-zoom", (zoom) => {
      this.set_zoom(zoom);
    });
    this.$root.$on("pan-to", (latitude, longitude, zoom = 12) => {
      this.pan_to(latitude, longitude, zoom);
    });
    this.$root.$on("show-shapes", (category, backRouter) => {
      this.load_category(category, backRouter);
    });
    this.$root.$on("show-markers", (category, backRouter) => {
      this.load_markers(category, backRouter);
    });
    this.$root.$on("reset-main-map", async () => {
      this.mainMapView = "all";
      this.activeSearchProjectIds = null;
      this.clearSearchProjectHighlight();
      await this.load_markers("all", ["home", "نظرة عامة"]);
    });
    this.$root.$on("show-search-results", (projectIds) => {
      this.showOnlySearchResults(projectIds);
    });
    this.$root.$on("highlight-search-project", (projectId) => {
      this.highlightSearchProject(projectId);
    });
    this.$root.$on("clear-search-project-highlight", () => {
      this.clearSearchProjectHighlight();
    });
    this.$root.$on("clear-search-results", () => {
      this.clearSearchProjectHighlight();
      this.activeSearchProjectIds = null;
      this.activeViewMode = "all";
      this.currentProjectId = null;
      this.removeThreeParkLayer();
      this.ensureConceptModeAllowed();
      this.renderMapData();
    });
    this.$root.$on("set-contract-map-filter", (filter) => {
      this.contractMapFilter = filter || "all";
      this.refreshMapForCurrentFilters();
    });
    this.$root.$on("set-project-status-map-filter", (filter) => {
      this.contractMapFilter = filter || "all";
      this.refreshMapForCurrentFilters();
    });
    this.$root.$on("show-project-marker", (item, index, toggle) => {
      this.load_project_marker(item, index, toggle);
    });
    this.$root.$on("show-shapes-districts", (category, backRouter) => {
      this.load_districts(category, backRouter);
    });
  },
  beforeDestroy() {
    this.stopMapboxLanguageSync();
    this.stopUserLocationTracking();

    if (this.locationMessageTimeout) {
      window.clearTimeout(this.locationMessageTimeout);
      this.locationMessageTimeout = null;
    }

    if (this.mapOverlayResizeObserver) {
      this.mapOverlayResizeObserver.disconnect();
      this.mapOverlayResizeObserver = null;
    }

    if (this.mapOverlayResizeHandler) {
      window.removeEventListener("resize", this.mapOverlayResizeHandler);
      this.mapOverlayResizeHandler = null;
    }

    if (this.mapExpandedKeydownHandler) {
      window.removeEventListener("keydown", this.mapExpandedKeydownHandler);
      this.mapExpandedKeydownHandler = null;
    }
    document.documentElement.classList.remove("green-riyadh-map-expanded");
    document.body.classList.remove("green-riyadh-map-expanded");

    this.disableMiddleMousePanNavigation();
    this.clearMapCameraRestoreTimers();
    this.clearSatelliteTerrainRestoreTimers();
    this.removeConceptMapLayers();
    this.destroyStandaloneThreeScene();

    if (this.$map) {
      this.$map.off("style.load", this.handleMapStyleLoad);
    }
    this.removeThreeParkLayer();

    if (this.mapPopup) {
      this.mapPopup.remove();
      this.mapPopup = null;
    }

    if (this.$map) {
      this.$map.remove();
      this.$map = null;
    }
  },
  methods: {
    isProjectDetailsRoute(route) {
      if (!route) return false;

      if (route.name === "projects") return true;

      const path = String(route.path || "");
      return /^\/projects\/[^/]+(?:\/|$)/i.test(path);
    },

    resetMapAfterProjectDetails() {
      // Always leave expanded mode first so the destination page gets its
      // normal layout and document scrolling back immediately.
      if (this.isMapExpanded) {
        this.setMapExpanded(false);
      } else {
        document.documentElement.classList.remove("green-riyadh-map-expanded");
        document.body.classList.remove("green-riyadh-map-expanded");
      }

      // Close ProjectDetails-only tools/panels and restore their defaults.
      this.$set(this.threeToolState, "structureLabels", false);
      this.$set(this.threeToolState, "conceptPaths", false);
      this.$set(this.threeToolState, "treeTypes", false);
      this.threeTreeTypeLoading = false;
      this.threeTreeTypeStats = [];
      this.threeStructureAnnotations = [];
      this.threeVisibleStructureLabels = [];
      this.resetStandaloneThreeLayerFilters();

      // Cancel the standalone Three.js project scene and discard its camera
      // state so it can never leak into another page.
      this.exitStandaloneThreeMode();
      this.removeConceptMapLayers();
      this.clearMapCameraRestoreTimers();
      this.clearSatelliteTerrainRestoreTimers();
      this.preservedMapViewState = null;
      this.pendingBaseMapCameraState = null;
      this.lastStandaloneThreeResetMapViewState = null;
      this.isBaseMapStyleSwitching = false;
      this.suspendMapStyleRestoration = false;
      this.mapboxRendererSuspended = false;

      // Clear the ProjectDetails identity before the next page mounts. Its own
      // load event can then establish the appropriate category/search view.
      this.currentProjectId = null;
      this.nearProjectId = null;
      this.nearProjectCenter = null;
      this.activeViewMode = "all";
      this.activeBaseMapMode = "street";

      if (!this.$map) return;

      this.showMapboxRenderer();

      // Reset the camera immediately. Mapbox setStyle() preserves the camera,
      // so doing this before restoring Street also avoids an old project/3D
      // camera being re-applied after the style finishes loading.
      try {
        this.$map.jumpTo({
          center: this.getDefaultCenter(),
          zoom: Number(this.zoom || 10),
          pitch: 0,
          bearing: 0,
        });
      } catch (error) {
        console.warn(
          "Map camera could not be reset while leaving ProjectDetails:",
          error,
        );
      }

      // ProjectDetails may have been in Satellite or standalone 3D. Force the
      // shared map back to the normal Street basemap for the destination page.
      this.projectLayersReady = false;
      this.unbindProjectLayerEvents();
      try {
        this.$map.setStyle(this.getMapboxStyleForMode("street"));
        this.$map.once("idle", () => {
          if (!this.$map || this.activeBaseMapMode !== "street") return;
          this.restoreMapOverlaysAfterStyleChange({ moveCamera: false });
        });
      } catch (error) {
        console.warn(
          "Street map style could not be restored while leaving ProjectDetails:",
          error,
        );
      }
    },

    getExpandedMapLeftOffset() {
      const viewportWidth = Math.max(window.innerWidth || 0, 1);
      const preferredSelectors = [
        ".home",
        ".main-content",
        ".content-wrapper",
        "main",
        "#app main",
      ];

      for (const selector of preferredSelectors) {
        const element = document.querySelector(selector);
        if (!element || element === this.$el || this.$el.contains(element))
          continue;
        const rect = element.getBoundingClientRect();
        if (rect.left > 0 && rect.left < viewportWidth * 0.35) {
          return Math.round(rect.left);
        }
      }

      const sidebarSelectors = [
        "#sidebar",
        ".sidebar",
        ".main-sidebar",
        ".left-sidebar",
        ".side-menu",
        ".sidebar-menu",
        "aside",
      ];
      for (const selector of sidebarSelectors) {
        const element = document.querySelector(selector);
        if (!element) continue;
        const rect = element.getBoundingClientRect();
        if (
          rect.left <= 2 &&
          rect.right > 0 &&
          rect.width < viewportWidth * 0.35 &&
          rect.height > window.innerHeight * 0.5
        ) {
          return Math.round(rect.right);
        }
      }

      return 0;
    },

    toggleMapExpanded() {
      this.setMapExpanded(!this.isMapExpanded);
    },

    setMapExpanded(expanded) {
      this.isMapExpanded = Boolean(expanded);
      this.mapExpandedLeft = this.isMapExpanded
        ? this.getExpandedMapLeftOffset()
        : 0;
      // Lock the document itself as well as the body so no browser scrollbar
      // remains visible while the map is using the expanded workspace.
      document.documentElement.classList.toggle(
        "green-riyadh-map-expanded",
        this.isMapExpanded,
      );
      document.body.classList.toggle(
        "green-riyadh-map-expanded",
        this.isMapExpanded,
      );

      this.$nextTick(() => {
        this.syncThreeParkHostBounds();
        this.resizeStandaloneThreeRenderer();
        this.updateMapOverlayControlPositions();
        if (this.$map && typeof this.$map.resize === "function") {
          this.$map.resize();
        }
        window.setTimeout(() => {
          this.syncThreeParkHostBounds();
          this.resizeStandaloneThreeRenderer();
          this.updateMapOverlayControlPositions();
          if (this.$map && typeof this.$map.resize === "function") {
            this.$map.resize();
          }
        }, 80);
      });
    },

    configureMapboxScrollZoomSensitivity() {
      if (!this.$map || !this.$map.scrollZoom) return;

      try {
        if (typeof this.$map.scrollZoom.enable === "function") {
          this.$map.scrollZoom.enable();
        }
        if (typeof this.$map.scrollZoom.setWheelZoomRate === "function") {
          this.$map.scrollZoom.setWheelZoomRate(MAPBOX_MOUSE_WHEEL_ZOOM_RATE);
        }
        if (typeof this.$map.scrollZoom.setZoomRate === "function") {
          this.$map.scrollZoom.setZoomRate(MAPBOX_TRACKPAD_ZOOM_RATE);
        }
      } catch (error) {
        console.warn(
          "Mapbox scroll zoom sensitivity could not be configured:",
          error,
        );
      }
    },

    enableMiddleMousePanNavigation() {
      if (!this.$map || this.middleMousePanHandlers) return;

      const canvas = this.$map.getCanvas && this.$map.getCanvas();
      if (!canvas) return;

      const preventMiddleClickAutoscroll = (event) => {
        if (event && event.button === 1) {
          event.preventDefault();
          if (typeof event.stopPropagation === "function")
            event.stopPropagation();
          if (typeof event.stopImmediatePropagation === "function") {
            event.stopImmediatePropagation();
          }
        }
      };

      const onMouseMove = (event) => {
        const state = this.middleMousePanState;
        if (!state || !state.active || !this.$map) return;

        event.preventDefault();
        if (typeof event.stopPropagation === "function")
          event.stopPropagation();
        if (typeof event.stopImmediatePropagation === "function") {
          event.stopImmediatePropagation();
        }

        const dx = event.clientX - state.lastX;
        const dy = event.clientY - state.lastY;

        state.lastX = event.clientX;
        state.lastY = event.clientY;

        if (dx || dy) {
          // Middle-mouse drag should behave like a flat hand-pan.
          // It intentionally preserves the current pitch and bearing so users
          // can follow roads/paths without accidentally rotating or tilting 3D view.
          this.$map.panBy([-dx, -dy], { duration: 0, animate: false });
          if (typeof this.$map.setPitch === "function")
            this.$map.setPitch(state.pitch);
          if (typeof this.$map.setBearing === "function")
            this.$map.setBearing(state.bearing);
        }
      };

      const finishMiddleMousePan = (event) => {
        const state = this.middleMousePanState;
        if (!state || !state.active) return;

        if (event) {
          event.preventDefault();
          if (typeof event.stopPropagation === "function")
            event.stopPropagation();
          if (typeof event.stopImmediatePropagation === "function") {
            event.stopImmediatePropagation();
          }
        }

        this.middleMousePanState = null;
        canvas.classList.remove("is-middle-mouse-panning");

        if (this.$map) {
          if (typeof this.$map.setPitch === "function")
            this.$map.setPitch(state.pitch);
          if (typeof this.$map.setBearing === "function")
            this.$map.setBearing(state.bearing);
          if (
            this.$map.dragRotate &&
            typeof this.$map.dragRotate.enable === "function"
          ) {
            this.$map.dragRotate.enable();
          }
        }

        window.removeEventListener("mousemove", onMouseMove, true);
        window.removeEventListener("mouseup", finishMiddleMousePan, true);
        window.removeEventListener("blur", finishMiddleMousePan, true);
      };

      const onMouseDown = (event) => {
        if (!event || event.button !== 1 || !this.$map) return;

        event.preventDefault();
        if (typeof event.stopPropagation === "function")
          event.stopPropagation();
        if (typeof event.stopImmediatePropagation === "function") {
          event.stopImmediatePropagation();
        }

        if (
          this.$map.dragRotate &&
          typeof this.$map.dragRotate.disable === "function"
        ) {
          this.$map.dragRotate.disable();
        }

        this.middleMousePanState = {
          active: true,
          lastX: event.clientX,
          lastY: event.clientY,
          pitch:
            typeof this.$map.getPitch === "function" ? this.$map.getPitch() : 0,
          bearing:
            typeof this.$map.getBearing === "function"
              ? this.$map.getBearing()
              : 0,
        };

        canvas.classList.add("is-middle-mouse-panning");

        window.addEventListener("mousemove", onMouseMove, true);
        window.addEventListener("mouseup", finishMiddleMousePan, true);
        window.addEventListener("blur", finishMiddleMousePan, true);
      };

      canvas.addEventListener("mousedown", onMouseDown, true);
      canvas.addEventListener("auxclick", preventMiddleClickAutoscroll, true);
      canvas.addEventListener(
        "contextmenu",
        preventMiddleClickAutoscroll,
        true,
      );

      this.middleMousePanHandlers = {
        canvas,
        onMouseDown,
        preventMiddleClickAutoscroll,
        onMouseMove,
        finishMiddleMousePan,
      };
    },

    disableMiddleMousePanNavigation() {
      const handlers = this.middleMousePanHandlers;
      if (!handlers) return;

      if (handlers.canvas) {
        handlers.canvas.removeEventListener(
          "mousedown",
          handlers.onMouseDown,
          true,
        );
        handlers.canvas.removeEventListener(
          "auxclick",
          handlers.preventMiddleClickAutoscroll,
          true,
        );
        handlers.canvas.removeEventListener(
          "contextmenu",
          handlers.preventMiddleClickAutoscroll,
          true,
        );
        handlers.canvas.classList.remove("is-middle-mouse-panning");
      }

      window.removeEventListener("mousemove", handlers.onMouseMove, true);
      window.removeEventListener(
        "mouseup",
        handlers.finishMiddleMousePan,
        true,
      );
      window.removeEventListener("blur", handlers.finishMiddleMousePan, true);

      if (
        this.$map &&
        this.$map.dragRotate &&
        typeof this.$map.dragRotate.enable === "function"
      ) {
        this.$map.dragRotate.enable();
      }

      this.middleMousePanState = null;
      this.middleMousePanHandlers = null;
    },

    updateMapOverlayControlPositions() {
      if (typeof window === "undefined" || typeof document === "undefined")
        return;

      this.$nextTick(() => {
        const mapElement = this.$refs && this.$refs.map;
        if (
          !mapElement ||
          typeof mapElement.getBoundingClientRect !== "function"
        )
          return;

        const rect = mapElement.getBoundingClientRect();
        const style = document.documentElement.style;
        const threeHost = this.$refs ? this.$refs.threeParkHost : null;
        const overlaySelector = [
          ".main-map-style-switch",
          ".three-tree-type-panel",
          ".three-layer-filters",
          ".main-map-layer-switch",
          ".map-navigation-controls",
          ".map-location-message",
          ".tse-map-legend--overlay",
        ].join(", ");
        const overlayElements = this.$el
          ? Array.from(this.$el.querySelectorAll(overlaySelector))
          : [];

        // Temporarily hide every fixed map control while checking which part
        // of the map is genuinely visible. This prevents the controls
        // themselves from being mistaken for page content covering the map.
        const previousVisibilities = overlayElements.map((element) =>
          element.style.getPropertyValue("visibility"),
        );
        overlayElements.forEach((element) => {
          element.style.setProperty("visibility", "hidden", "important");
        });

        let visibleLeft = Math.max(0, Math.floor(rect.left));
        let visibleTop = Math.max(0, Math.floor(rect.top));
        let visibleRight = Math.min(window.innerWidth, Math.ceil(rect.right));
        let visibleBottom = Math.min(
          window.innerHeight,
          Math.ceil(rect.bottom),
        );
        let foundVisibleMapPoint = false;

        try {
          const startX = visibleLeft;
          const endX = Math.max(
            startX,
            Math.min(window.innerWidth - 1, visibleRight - 1),
          );
          const startY = visibleTop;
          const endY = Math.max(
            startY,
            Math.min(window.innerHeight - 1, visibleBottom - 1),
          );
          const step = 10;
          let minVisibleX = Infinity;
          let minVisibleY = Infinity;
          let maxVisibleX = -Infinity;
          let maxVisibleY = -Infinity;

          for (let y = startY; y <= endY; y += step) {
            for (let x = startX; x <= endX; x += step) {
              const elementAtPoint = document.elementFromPoint(x, y);
              const pointIsMap =
                elementAtPoint &&
                (elementAtPoint === mapElement ||
                  mapElement.contains(elementAtPoint) ||
                  (this.isStandaloneThreeMode &&
                    threeHost &&
                    (elementAtPoint === threeHost ||
                      threeHost.contains(elementAtPoint))));

              if (!pointIsMap) continue;

              foundVisibleMapPoint = true;
              minVisibleX = Math.min(minVisibleX, x);
              minVisibleY = Math.min(minVisibleY, y);
              maxVisibleX = Math.max(maxVisibleX, x);
              maxVisibleY = Math.max(maxVisibleY, y);
            }
          }

          if (foundVisibleMapPoint) {
            visibleLeft = Math.max(0, minVisibleX);
            visibleTop = Math.max(0, minVisibleY);
            visibleRight = Math.min(
              window.innerWidth,
              Math.min(rect.right, maxVisibleX + step),
            );
            visibleBottom = Math.min(
              window.innerHeight,
              Math.min(rect.bottom, maxVisibleY + step),
            );
          }
        } finally {
          overlayElements.forEach((element, index) => {
            const previousVisibility = previousVisibilities[index];
            if (previousVisibility) {
              element.style.setProperty("visibility", previousVisibility);
            } else {
              element.style.removeProperty("visibility");
            }
          });
        }

        // If the responsive page completely covers the map, hide its fixed
        // controls too. They will be restored automatically on the next
        // resize/Map resize when a visible map area exists again.
        if (!foundVisibleMapPoint) {
          overlayElements.forEach((element) => {
            element.classList.add("is-outside-visible-map");
          });
          return;
        }

        const edgePadding = 14;
        const top = Math.max(edgePadding, Math.round(visibleTop + edgePadding));
        const left = Math.max(
          edgePadding,
          Math.round(visibleLeft + edgePadding),
        );
        // Use the actual #map element right edge for right-anchored controls.
        // The visibility scan is still useful for the left/top/bottom bounds,
        // but on laptop layouts it can underestimate the visible right edge
        // and push Layer Filters too far inward.
        const mapRight = Math.max(
          visibleLeft,
          Math.min(window.innerWidth, Math.ceil(rect.right)),
        );
        const mapBottom = Math.max(visibleTop, visibleBottom);
        const navBottom = Math.max(
          70,
          Math.round(window.innerHeight - mapBottom + 70),
        );
        const overlayMaxHeight = Math.max(
          0,
          Math.floor(mapBottom - top - edgePadding),
        );
        const treePanelMaxHeight = Math.max(
          0,
          Math.floor(mapBottom - (top + 52) - edgePadding),
        );
        const styleMaxWidth = Math.max(
          0,
          Math.floor(mapRight - left - edgePadding),
        );

        const switchElement = this.$el
          ? this.$el.querySelector(".main-map-style-switch")
          : null;
        const styleSwitchRect = switchElement
          ? switchElement.getBoundingClientRect()
          : null;
        const styleSwitchWidth = styleSwitchRect ? styleSwitchRect.width : 0;

        // Keep Layer Filters visually symmetrical with the Street / Satellite /
        // 3D selector. The selector is 14px from the viewport's left edge, so
        // Layer Filters must be 14px from the viewport's right edge. Do not
        // derive this from #map.right here: #map can intentionally be 32%/64%
        // wide in the page layout, which created the large empty strip to the
        // right of the filter panel in 3D mode.
        const filterRight = edgePadding;

        const naturalLayerWidth = window.innerWidth <= 767 ? 214 : 244;
        const panelGap = 10;
        const filterRightEdge = Math.max(
          visibleLeft,
          Math.round(mapRight - edgePadding),
        );
        const styleSwitchRight = left + styleSwitchWidth;
        const availableLayerWidth = Math.max(
          0,
          Math.floor(filterRightEdge - styleSwitchRight - panelGap),
        );

        // Never fall back to the natural width when available width is 0.
        // The old `availableLayerWidth || naturalLayerWidth` fallback was the
        // reason the panels could still overlap on narrow laptop map columns.
        const layerFilterWidth = Math.max(
          0,
          Math.min(naturalLayerWidth, availableLayerWidth),
        );

        style.setProperty(
          "--green-riyadh-map-control-top",
          `${Math.round(top)}px`,
        );
        style.setProperty(
          "--green-riyadh-map-control-left",
          `${Math.round(left)}px`,
        );
        // Layer Filters uses the same top inset as the Street / Satellite /
        // 3D controls and the exact same 14px inset from the visible map edge.
        style.setProperty(
          "--green-riyadh-map-filter-top",
          `${Math.round(top)}px`,
        );
        style.setProperty(
          "--green-riyadh-map-filter-right",
          `${filterRight}px`,
        );
        style.setProperty("--green-riyadh-map-nav-bottom", `${navBottom}px`);
        style.setProperty(
          "--green-riyadh-map-overlay-max-height",
          `${overlayMaxHeight}px`,
        );
        style.setProperty(
          "--green-riyadh-map-tree-panel-max-height",
          `${treePanelMaxHeight}px`,
        );
        style.setProperty(
          "--green-riyadh-map-style-max-width",
          `${styleMaxWidth}px`,
        );
        style.setProperty(
          "--green-riyadh-map-layer-filter-width",
          `${Math.max(0, layerFilterWidth)}px`,
        );

        // Keep every fixed panel/control completely inside the portion of the
        // map that is actually visible. On very small browser windows a panel
        // that cannot fit is hidden instead of floating over ProjectDetails.
        overlayElements.forEach((element) => {
          const elementRect = element.getBoundingClientRect();
          const fitsVisibleMap =
            elementRect.width > 0 &&
            elementRect.height > 0 &&
            elementRect.left >= visibleLeft - 1 &&
            elementRect.right <= mapRight + 1 &&
            elementRect.top >= visibleTop - 1 &&
            elementRect.bottom <= mapBottom + 1;

          element.classList.toggle("is-outside-visible-map", !fitsVisibleMap);
        });
      });
    },

    getAppLanguageCode() {
      const htmlElement =
        typeof document !== "undefined" ? document.documentElement : null;
      const storage =
        typeof window !== "undefined" ? window.localStorage : null;
      const possibleValues = [
        htmlElement ? htmlElement.lang : "",
        htmlElement && htmlElement.dir === "rtl" ? "ar" : "",
        this.$i18n?.locale,
        this.$root?.$i18n?.locale,
        this.$root?.$data?.locale,
        this.$root?.$data?.lang,
        this.$root?.$data?.language,
        storage ? storage.getItem("locale") : "",
        storage ? storage.getItem("lang") : "",
        storage ? storage.getItem("language") : "",
        storage ? storage.getItem("i18n_locale") : "",
      ];

      const matchedValue = possibleValues.find((value) => {
        const text = String(value || "")
          .trim()
          .toLowerCase();
        return text.startsWith("ar") || text.startsWith("en") || text === "rtl";
      });

      const text = String(matchedValue || "")
        .trim()
        .toLowerCase();
      return text.startsWith("ar") || text === "rtl"
        ? MAPBOX_LANGUAGE_ARABIC
        : MAPBOX_LANGUAGE_ENGLISH;
    },

    getMapboxLanguage() {
      return this.getAppLanguageCode() === MAPBOX_LANGUAGE_ARABIC
        ? MAPBOX_LANGUAGE_ARABIC
        : MAPBOX_LANGUAGE_ENGLISH;
    },

    applyMapboxLabelLanguage(force = false) {
      if (!this.$map || typeof this.$map.setLanguage !== "function") return;

      const nextLanguage = this.getMapboxLanguage();
      if (!force && this.currentMapboxLanguage === nextLanguage) return;

      this.currentMapboxLanguage = nextLanguage;

      try {
        this.$map.setLanguage(nextLanguage);
      } catch (error) {
        console.warn("Mapbox label language could not be updated:", error);
      }
    },

    startMapboxLanguageSync() {
      if (typeof window === "undefined" || typeof document === "undefined")
        return;

      this.applyMapboxLabelLanguage(true);

      const syncLanguage = () => {
        this.applyMapboxLabelLanguage(false);
      };

      if (
        typeof MutationObserver !== "undefined" &&
        !this.mapboxLanguageObserver
      ) {
        this.mapboxLanguageObserver = new MutationObserver(syncLanguage);
        this.mapboxLanguageObserver.observe(document.documentElement, {
          attributes: true,
          attributeFilter: ["lang", "dir", "class"],
        });
        if (document.body) {
          this.mapboxLanguageObserver.observe(document.body, {
            attributes: true,
            attributeFilter: ["lang", "dir", "class"],
          });
        }
      }

      const rootEvents = [
        "change-language",
        "change-lang",
        "language-changed",
        "locale-changed",
        "set-language",
        "set-locale",
      ];
      this.mapboxLanguageRootHandlers = rootEvents.map((eventName) => {
        const handler = () => this.$nextTick(syncLanguage);
        if (this.$root && typeof this.$root.$on === "function") {
          this.$root.$on(eventName, handler);
        }
        return { eventName, handler };
      });

      if (!this.mapboxLanguageSyncInterval) {
        this.mapboxLanguageSyncInterval = window.setInterval(
          syncLanguage,
          1000,
        );
      }
    },

    stopMapboxLanguageSync() {
      if (this.mapboxLanguageObserver) {
        this.mapboxLanguageObserver.disconnect();
        this.mapboxLanguageObserver = null;
      }

      if (this.mapboxLanguageSyncInterval) {
        window.clearInterval(this.mapboxLanguageSyncInterval);
        this.mapboxLanguageSyncInterval = null;
      }

      if (
        Array.isArray(this.mapboxLanguageRootHandlers) &&
        this.$root &&
        typeof this.$root.$off === "function"
      ) {
        this.mapboxLanguageRootHandlers.forEach(({ eventName, handler }) => {
          this.$root.$off(eventName, handler);
        });
      }
      this.mapboxLanguageRootHandlers = [];
    },

    hasMapbox() {
      return !!(
        this.mapboxgl &&
        this.$map &&
        typeof this.$map.getZoom === "function"
      );
    },

    async ensureMapboxLoaded() {
      if (!isMapboxTokenConfigured(MAPBOX_ACCESS_TOKEN)) {
        throw new Error(
          "Missing Mapbox public token. Add it to MAPBOX_ACCESS_TOKEN or VUE_APP_MAPBOX_ACCESS_TOKEN.",
        );
      }

      this.mapboxgl = await loadMapboxGL();
      this.mapboxgl.accessToken = MAPBOX_ACCESS_TOKEN.trim();
      ensureMapboxRtlTextPlugin(this.mapboxgl);
      return this.mapboxgl;
    },

    async waitForMapReady() {
      try {
        await this.ensureMapboxLoaded();
        if (!this.$map) {
          await this.init_map();
        }
        if (this.mapReadyPromise) {
          await this.mapReadyPromise;
        }
        return !!this.$map;
      } catch (error) {
        console.error("Map is not ready:", error);
        return false;
      }
    },

    async setMainMapView(view) {
      const filter = this.mainMapFilters.find((item) => item.value === view);
      if (!filter) return;

      this.mainMapView = filter.value;
      this.activeSearchProjectIds = null;
      this.clearSearchProjectHighlight();

      await this.load_markers(filter.category, [
        "home",
        this.mapUiText("Overview", "نظرة عامة", "Overview"),
      ]);
    },

    getMainMapViewByCategory(category) {
      const normalizedCategory = category || "all";
      const filter = this.mainMapFilters.find(
        (item) => item.category === normalizedCategory,
      );

      if (filter) return filter.value;
      return this.isTseTypology(normalizedCategory) ? "tse" : "all";
    },

    getDefaultCenter() {
      return [Number(this.longitude), Number(this.latitude)];
    },

    getMapboxStyleForMode(mode = this.activeBaseMapMode, theme = this.theme) {
      if (mode === "satellite") return MAPBOX_STYLE_SATELLITE;
      if (this.isThreeDimensionalMapMode(mode)) return MAPBOX_STYLE_3D;
      return theme === "dark" ? MAPBOX_STYLE_DARK : MAPBOX_STYLE_LIGHT;
    },

    isMapboxRoadLayer(layer) {
      if (!layer || layer.type !== "line" || !layer.id) return false;

      const id = String(layer.id).toLowerCase();
      const sourceId = String(layer.source || "").toLowerCase();
      const sourceLayer = String(layer["source-layer"] || "").toLowerCase();

      // Never touch Green Riyadh project/TSE/district layers. Only change the
      // Mapbox basemap road layers so the orange project geometry is not
      // confused with orange highway/major-road styling from the basemap.
      if (
        id.includes("green-riyadh") ||
        sourceId === PROJECT_SOURCE_ID ||
        sourceId === DISTRICT_SOURCE_ID
      ) {
        return false;
      }

      return (
        sourceLayer === "road" ||
        sourceLayer.includes("road") ||
        id.includes("road") ||
        id.includes("motorway") ||
        id.includes("trunk") ||
        id.includes("highway") ||
        id.includes("bridge") ||
        id.includes("tunnel")
      );
    },

    isMapboxRoadSymbolLayer(layer) {
      if (!layer || layer.type !== "symbol" || !layer.id) return false;

      const id = String(layer.id).toLowerCase();
      const sourceId = String(layer.source || "").toLowerCase();
      const sourceLayer = String(layer["source-layer"] || "").toLowerCase();

      // Keep Green Riyadh custom labels/icons untouched.
      if (
        id.includes("green-riyadh") ||
        sourceId === PROJECT_SOURCE_ID ||
        sourceId === DISTRICT_SOURCE_ID
      ) {
        return false;
      }

      return (
        sourceLayer === "road" ||
        sourceLayer.includes("road") ||
        id.includes("road") ||
        id.includes("motorway") ||
        id.includes("trunk") ||
        id.includes("highway") ||
        id.includes("route") ||
        id.includes("shield")
      );
    },

    isMapboxGreenAreaLayer(layer) {
      if (!layer || !layer.id) return false;
      if (!["fill", "line"].includes(layer.type)) return false;

      const id = String(layer.id).toLowerCase();
      const sourceId = String(layer.source || "").toLowerCase();
      const sourceLayer = String(layer["source-layer"] || "").toLowerCase();

      // Never touch Green Riyadh project/TSE/district layers. Only neutralize
      // Mapbox basemap parks/green-area layers so they do not look like
      // completed Green Riyadh project geometry.
      if (
        id.includes("green-riyadh") ||
        sourceId === PROJECT_SOURCE_ID ||
        sourceId === DISTRICT_SOURCE_ID
      ) {
        return false;
      }

      if (
        id.includes("water") ||
        sourceLayer.includes("water") ||
        id.includes("building") ||
        sourceLayer.includes("building") ||
        this.isMapboxRoadLayer(layer)
      ) {
        return false;
      }

      const greenAreaKeywords = [
        "park",
        "landuse",
        "land-use",
        "landcover",
        "land-cover",
        "natural",
        "wood",
        "forest",
        "grass",
        "vegetation",
        "greenspace",
        "green-space",
        "golf",
        "pitch",
        "recreation",
        "cemetery",
        "garden",
      ];

      return greenAreaKeywords.some(
        (keyword) => id.includes(keyword) || sourceLayer.includes(keyword),
      );
    },

    applyNeutralBaseRoadStyle() {
      if (!this.$map || !this.$map.isStyleLoaded()) return;

      const style = this.$map.getStyle && this.$map.getStyle();
      const layers = Array.isArray(style?.layers) ? style.layers : [];
      const isDark =
        this.theme === "dark" && this.activeBaseMapMode === "street";
      const roadColor = isDark
        ? MAPBOX_NEUTRAL_ROAD_COLOR_DARK
        : MAPBOX_NEUTRAL_ROAD_COLOR_LIGHT;
      const roadOutlineColor = isDark
        ? MAPBOX_NEUTRAL_ROAD_OUTLINE_DARK
        : MAPBOX_NEUTRAL_ROAD_OUTLINE_LIGHT;
      const roadOpacity = isDark
        ? MAPBOX_NEUTRAL_ROAD_OPACITY_DARK
        : MAPBOX_NEUTRAL_ROAD_OPACITY_LIGHT;
      const roadOutlineOpacity = isDark
        ? MAPBOX_NEUTRAL_ROAD_OUTLINE_OPACITY_DARK
        : MAPBOX_NEUTRAL_ROAD_OUTLINE_OPACITY_LIGHT;
      const roadLabelColor = isDark
        ? MAPBOX_NEUTRAL_ROAD_LABEL_DARK
        : MAPBOX_NEUTRAL_ROAD_LABEL_LIGHT;
      const roadLabelOpacity = isDark
        ? MAPBOX_NEUTRAL_ROAD_LABEL_OPACITY_DARK
        : MAPBOX_NEUTRAL_ROAD_LABEL_OPACITY_LIGHT;
      const roadShieldOpacity = isDark
        ? MAPBOX_NEUTRAL_ROAD_SHIELD_OPACITY_DARK
        : MAPBOX_NEUTRAL_ROAD_SHIELD_OPACITY_LIGHT;
      const parkFillColor = isDark
        ? MAPBOX_NEUTRAL_PARK_FILL_DARK
        : MAPBOX_NEUTRAL_PARK_FILL_LIGHT;
      const parkOutlineColor = isDark
        ? MAPBOX_NEUTRAL_PARK_OUTLINE_DARK
        : MAPBOX_NEUTRAL_PARK_OUTLINE_LIGHT;
      const parkFillOpacity = isDark
        ? MAPBOX_NEUTRAL_PARK_FILL_OPACITY_DARK
        : MAPBOX_NEUTRAL_PARK_FILL_OPACITY_LIGHT;
      const parkLineOpacity = isDark
        ? MAPBOX_NEUTRAL_PARK_LINE_OPACITY_DARK
        : MAPBOX_NEUTRAL_PARK_LINE_OPACITY_LIGHT;

      layers.forEach((layer) => {
        const layerId = layer.id;
        const id = String(layerId).toLowerCase();

        if (this.isMapboxRoadLayer(layer)) {
          const isOutlineLayer =
            id.includes("case") ||
            id.includes("casing") ||
            id.includes("outline") ||
            id.includes("border");
          const neutralColor = isOutlineLayer ? roadOutlineColor : roadColor;
          const neutralOpacity = isOutlineLayer
            ? roadOutlineOpacity
            : roadOpacity;

          try {
            this.$map.setPaintProperty(layerId, "line-color", neutralColor);
            this.$map.setPaintProperty(layerId, "line-opacity", neutralOpacity);
          } catch (error) {
            // Some imported Mapbox Standard layers may not allow this paint update.
          }

          return;
        }

        if (this.isMapboxRoadSymbolLayer(layer)) {
          const isShieldLayer =
            id.includes("shield") ||
            id.includes("number") ||
            id.includes("route");

          try {
            this.$map.setPaintProperty(layerId, "text-color", roadLabelColor);
            this.$map.setPaintProperty(
              layerId,
              "text-halo-color",
              isDark ? "#20272b" : "#f7f7f4",
            );
            this.$map.setPaintProperty(
              layerId,
              "text-opacity",
              isShieldLayer ? 0.5 : roadLabelOpacity,
            );
            this.$map.setPaintProperty(
              layerId,
              "icon-opacity",
              isShieldLayer ? roadShieldOpacity : 0.45,
            );
          } catch (error) {
            // Some imported Mapbox Standard layers may not allow this paint update.
          }

          return;
        }

        if (this.isMapboxGreenAreaLayer(layer)) {
          try {
            if (layer.type === "fill") {
              this.$map.setPaintProperty(layerId, "fill-color", parkFillColor);
              this.$map.setPaintProperty(
                layerId,
                "fill-outline-color",
                parkOutlineColor,
              );
              this.$map.setPaintProperty(
                layerId,
                "fill-opacity",
                parkFillOpacity,
              );
            } else if (layer.type === "line") {
              this.$map.setPaintProperty(
                layerId,
                "line-color",
                parkOutlineColor,
              );
              this.$map.setPaintProperty(
                layerId,
                "line-opacity",
                parkLineOpacity,
              );
            }
          } catch (error) {
            // Some imported Mapbox Standard layers may not allow this paint update.
          }
        }
      });
    },

    unbindProjectLayerEvents() {
      if (!this.$map || !this.mapLayerEventsBound) return;

      const layerHandlers = [
        [PROJECT_FILL_LAYER_ID, "mouseenter", this.onProjectMouseEnter],
        [PROJECT_LINE_LAYER_ID, "mouseenter", this.onProjectMouseEnter],
        [PROJECT_FACILITY_LAYER_ID, "mouseenter", this.onProjectMouseEnter],
        [PROJECT_FILL_LAYER_ID, "mousemove", this.onProjectMouseMove],
        [PROJECT_LINE_LAYER_ID, "mousemove", this.onProjectMouseMove],
        [PROJECT_FACILITY_LAYER_ID, "mousemove", this.onProjectMouseMove],
        [PROJECT_FILL_LAYER_ID, "mouseleave", this.onProjectMouseLeave],
        [PROJECT_LINE_LAYER_ID, "mouseleave", this.onProjectMouseLeave],
        [PROJECT_FACILITY_LAYER_ID, "mouseleave", this.onProjectMouseLeave],
        [PROJECT_FILL_LAYER_ID, "click", this.onProjectClick],
        [PROJECT_LINE_LAYER_ID, "click", this.onProjectClick],
        [PROJECT_FACILITY_LAYER_ID, "click", this.onProjectClick],
      ];

      layerHandlers.forEach(([layerId, eventName, handler]) => {
        try {
          this.$map.off(eventName, layerId, handler);
        } catch (error) {
          // The old style may have already removed the layer. That is fine.
        }
      });

      this.mapLayerEventsBound = false;
    },

    async handleMapStyleLoad() {
      if (this.suspendMapStyleRestoration) return;
      this.configureMapboxScrollZoomSensitivity();
      this.unbindProjectLayerEvents();
      await this.restoreMapOverlaysAfterStyleChange();
    },

    shouldShowThreeParkLayer() {
      return this.isStandaloneThreeMode;
    },

    getSelectedProjectCenter() {
      if (this.activeViewMode !== "project" || !this.currentProjectId) {
        return null;
      }

      const projectItem = this.getProjectItemById(this.currentProjectId);
      const savedCenter = this.getLngLatFromCenterValue(projectItem?.center);

      if (
        Array.isArray(savedCenter) &&
        savedCenter.length >= 2 &&
        savedCenter.every((value) => Number.isFinite(Number(value)))
      ) {
        return [Number(savedCenter[0]), Number(savedCenter[1])];
      }

      const projectFeatures = this.getProjectFeaturesForId(
        this.currentProjectId,
      );
      const bounds = this.getLngLatBoundsFromFeatures(projectFeatures);
      if (!bounds) return null;

      return [
        (Number(bounds.minLng) + Number(bounds.maxLng)) / 2,
        (Number(bounds.minLat) + Number(bounds.maxLat)) / 2,
      ];
    },

    normalizeTerrainRing(ring = []) {
      const normalized = (Array.isArray(ring) ? ring : [])
        .map((coordinate) => [
          Number(coordinate && coordinate[0]),
          Number(coordinate && coordinate[1]),
        ])
        .filter((coordinate) => coordinate.every(Number.isFinite));

      if (normalized.length > 1) {
        const first = normalized[0];
        const last = normalized[normalized.length - 1];
        if (
          Math.abs(first[0] - last[0]) < 1e-10 &&
          Math.abs(first[1] - last[1]) < 1e-10
        ) {
          normalized.pop();
        }
      }

      return normalized;
    },

    getProjectTerrainPolygonParts() {
      if (!this.currentProjectId) return [];

      const parts = [];
      const features = this.getProjectFeaturesForId(this.currentProjectId);

      const addPolygon = (polygonCoordinates) => {
        if (!Array.isArray(polygonCoordinates) || !polygonCoordinates.length) {
          return;
        }

        const outer = this.normalizeTerrainRing(polygonCoordinates[0]);
        if (outer.length < 3) return;

        const holes = polygonCoordinates
          .slice(1)
          .map((ring) => this.normalizeTerrainRing(ring))
          .filter((ring) => ring.length >= 3);

        parts.push({ outer, holes });
      };

      features.forEach((feature) => {
        const geometry = feature && feature.geometry;
        if (!geometry) return;

        if (geometry.type === "Polygon") {
          addPolygon(geometry.coordinates);
        } else if (geometry.type === "MultiPolygon") {
          geometry.coordinates.forEach(addPolygon);
        }
      });

      return parts;
    },

    getTerrainPartsBounds(parts = []) {
      const bounds = {
        minLng: Infinity,
        minLat: Infinity,
        maxLng: -Infinity,
        maxLat: -Infinity,
      };
      let hasCoordinates = false;

      parts.forEach((part) => {
        [part.outer, ...(part.holes || [])].forEach((ring) => {
          ring.forEach((coordinate) => {
            const lng = Number(coordinate[0]);
            const lat = Number(coordinate[1]);
            if (!Number.isFinite(lng) || !Number.isFinite(lat)) return;
            bounds.minLng = Math.min(bounds.minLng, lng);
            bounds.minLat = Math.min(bounds.minLat, lat);
            bounds.maxLng = Math.max(bounds.maxLng, lng);
            bounds.maxLat = Math.max(bounds.maxLat, lat);
            hasCoordinates = true;
          });
        });
      });

      return hasCoordinates ? bounds : null;
    },

    getTerrainOrigin(bounds) {
      if (!bounds) return null;
      return [
        (Number(bounds.minLng) + Number(bounds.maxLng)) / 2,
        (Number(bounds.minLat) + Number(bounds.maxLat)) / 2,
      ];
    },

    lngLatToTerrainMeters(coordinate, origin) {
      const lng = Number(coordinate[0]);
      const lat = Number(coordinate[1]);
      const originLng = Number(origin[0]);
      const originLat = Number(origin[1]);
      const latitudeScale = 111320;
      const longitudeScale =
        latitudeScale * Math.max(0.15, Math.cos((originLat * Math.PI) / 180));

      return {
        x: (lng - originLng) * longitudeScale,
        z: -(lat - originLat) * latitudeScale,
      };
    },

    terrainMetersToLngLat(point, origin) {
      const originLng = Number(origin[0]);
      const originLat = Number(origin[1]);
      const latitudeScale = 111320;
      const longitudeScale =
        latitudeScale * Math.max(0.15, Math.cos((originLat * Math.PI) / 180));

      return [
        originLng + Number(point.x) / longitudeScale,
        originLat - Number(point.z) / latitudeScale,
      ];
    },

    getTerrainPointKey(point) {
      return `${Number(point.x).toFixed(3)}|${Number(point.z).toFixed(3)}`;
    },

    getTerrainSubdivisionLevel(faceCount, maxDimension) {
      const safeFaceCount = Math.max(1, Number(faceCount) || 1);
      const byTriangleBudget = Math.floor(
        Math.sqrt(THREE_PARK_MAX_TRIANGLES / safeFaceCount),
      );
      const byPhysicalSize = Math.ceil(
        Math.max(1, Number(maxDimension) || 1) / 35,
      );

      return Math.max(
        THREE_PARK_MIN_SUBDIVISIONS,
        Math.min(THREE_PARK_MAX_SUBDIVISIONS, byTriangleBudget, byPhysicalSize),
      );
    },

    densifyTerrainRing(ring = [], maximumSegmentLength = 24) {
      if (!Array.isArray(ring) || ring.length < 2) return ring || [];

      const safeSegmentLength = Math.max(1, Number(maximumSegmentLength) || 24);
      const denseRing = [];

      for (let index = 0; index < ring.length; index += 1) {
        const current = ring[index];
        const next = ring[(index + 1) % ring.length];
        const dx = Number(next.x) - Number(current.x);
        const dz = Number(next.z) - Number(current.z);
        const distance = Math.sqrt(dx * dx + dz * dz);
        const steps = Math.max(1, Math.ceil(distance / safeSegmentLength));

        for (let step = 0; step < steps; step += 1) {
          const t = step / steps;
          denseRing.push({
            x: Number(current.x) + dx * t,
            z: Number(current.z) + dz * t,
          });
        }
      }

      return denseRing;
    },

    getStandaloneSatelliteTextureDimensions(bounds) {
      const centerLatitude =
        (Number(bounds.minLat) + Number(bounds.maxLat)) / 2;
      const widthMeters =
        (Number(bounds.maxLng) - Number(bounds.minLng)) *
        111320 *
        Math.max(0.15, Math.cos((centerLatitude * Math.PI) / 180));
      const heightMeters =
        (Number(bounds.maxLat) - Number(bounds.minLat)) * 111320;
      const safeWidth = Math.max(1, widthMeters);
      const safeHeight = Math.max(1, heightMeters);
      const maximum = THREE_PARK_STATIC_TEXTURE_MAX_DIMENSION;
      const minimum = THREE_PARK_STATIC_TEXTURE_MIN_DIMENSION;

      if (safeWidth >= safeHeight) {
        return {
          width: maximum,
          height: Math.max(
            minimum,
            Math.min(maximum, Math.round(maximum * (safeHeight / safeWidth))),
          ),
        };
      }

      return {
        width: Math.max(
          minimum,
          Math.min(maximum, Math.round(maximum * (safeWidth / safeHeight))),
        ),
        height: maximum,
      };
    },

    getMapboxSatelliteTextureUrl(bounds) {
      if (!bounds || !isMapboxTokenConfigured(MAPBOX_ACCESS_TOKEN)) return "";

      const dimensions = this.getStandaloneSatelliteTextureDimensions(bounds);
      const bbox = [
        Number(bounds.minLng).toFixed(6),
        Number(bounds.minLat).toFixed(6),
        Number(bounds.maxLng).toFixed(6),
        Number(bounds.maxLat).toFixed(6),
      ].join(",");

      // The Static Images API currently does not support Mapbox Standard
      // Satellite, so use the raster-only satellite-v9 style for the terrain
      // color texture. The interactive Satellite button still uses Standard
      // Satellite with 3D buildings and terrain.
      return (
        `https://api.mapbox.com/styles/v1/mapbox/satellite-v9/static/` +
        `[${bbox}]/${dimensions.width}x${dimensions.height}@2x` +
        `?logo=false&attribution=false&padding=0` +
        `&access_token=${encodeURIComponent(MAPBOX_ACCESS_TOKEN)}`
      );
    },

    loadStandaloneSatelliteTexture(bounds) {
      const url = this.getMapboxSatelliteTextureUrl(bounds);
      if (!url) return Promise.resolve(null);

      return new Promise((resolve) => {
        const loader = new THREE.TextureLoader();
        if (typeof loader.setCrossOrigin === "function") {
          loader.setCrossOrigin("anonymous");
        }

        let settled = false;
        let timer = null;
        const finish = (texture) => {
          if (settled) {
            if (texture && texture.dispose) texture.dispose();
            return;
          }
          settled = true;
          window.clearTimeout(timer);
          resolve(texture || null);
        };

        timer = window.setTimeout(() => {
          console.warn(
            "The Mapbox satellite texture timed out; using the dark terrain fallback.",
          );
          finish(null);
        }, THREE_PARK_STATIC_TEXTURE_TIMEOUT_MS);

        loader.load(
          url,
          (texture) => {
            texture.wrapS = THREE.ClampToEdgeWrapping;
            texture.wrapT = THREE.ClampToEdgeWrapping;
            texture.magFilter = THREE.LinearFilter;
            texture.minFilter = THREE.LinearMipmapLinearFilter;
            texture.generateMipmaps = true;
            if (THREE.SRGBColorSpace && "colorSpace" in texture) {
              texture.colorSpace = THREE.SRGBColorSpace;
            } else if (THREE.sRGBEncoding) {
              texture.encoding = THREE.sRGBEncoding;
            }
            texture.needsUpdate = true;
            finish(texture);
          },
          undefined,
          (error) => {
            console.warn(
              "The Mapbox satellite texture could not be loaded; using the dark terrain fallback.",
              error,
            );
            finish(null);
          },
        );
      });
    },

    loadStandaloneCompactedGroundTexture() {
      return new Promise((resolve) => {
        const loader = new THREE.TextureLoader();
        let settled = false;
        let timer = null;

        const finish = (texture) => {
          if (settled) {
            if (texture && texture.dispose) texture.dispose();
            return;
          }
          settled = true;
          window.clearTimeout(timer);
          resolve(texture || null);
        };

        timer = window.setTimeout(() => {
          console.warn(
            "The SEALED GROUND texture timed out. Check /public/images/sealed_ground.png.",
          );
          finish(null);
        }, THREE_PARK_STATIC_TEXTURE_TIMEOUT_MS);

        loader.load(
          THREE_PARK_COMPACTED_GROUND_TEXTURE_URL,
          (texture) => {
            // Repeat the material instead of stretching one photo across the
            // full park. The shader controls the repeat count from the terrain
            // size so the soil/stone scale stays believable.
            texture.wrapS = THREE.RepeatWrapping;
            texture.wrapT = THREE.RepeatWrapping;
            texture.magFilter = THREE.LinearFilter;
            texture.minFilter = THREE.LinearMipmapLinearFilter;
            texture.generateMipmaps = true;
            if (THREE.SRGBColorSpace && "colorSpace" in texture) {
              texture.colorSpace = THREE.SRGBColorSpace;
            } else if (THREE.sRGBEncoding) {
              texture.encoding = THREE.sRGBEncoding;
            }
            texture.needsUpdate = true;
            finish(texture);
          },
          undefined,
          (error) => {
            console.warn(
              "The SEALED GROUND texture could not be loaded. Check /public/images/sealed_ground.png.",
              error,
            );
            finish(null);
          },
        );
      });
    },

    createStandaloneFallbackTexture() {
      const pixel = new Uint8Array([2, 8, 11, 255]);
      const texture = new THREE.DataTexture(pixel, 1, 1, THREE.RGBAFormat);
      texture.needsUpdate = true;
      return texture;
    },

    getBarycentricTerrainPoint(a, b, c, u, v) {
      return {
        x: a.x + (b.x - a.x) * u + (c.x - a.x) * v,
        z: a.z + (b.z - a.z) * u + (c.z - a.z) * v,
      };
    },

    waitForMapEvent(eventName, timeout = THREE_PARK_IDLE_TIMEOUT_MS) {
      if (!this.$map) return Promise.resolve(false);

      return new Promise((resolve) => {
        let finished = false;
        const complete = (value) => {
          if (finished) return;
          finished = true;
          window.clearTimeout(timer);
          resolve(value);
        };
        const timer = window.setTimeout(() => complete(false), timeout);
        this.$map.once(eventName, () => complete(true));
      });
    },

    showMapboxRenderer() {
      const mapElement = this.$refs.map;
      if (mapElement) {
        mapElement.style.visibility = "visible";
        mapElement.style.pointerEvents = "auto";
      }
      if (this.$map && typeof this.$map.resize === "function") {
        this.$map.resize();
      }
    },

    async ensureTerrainSamplingReady(bounds) {
      if (!this.$map || !this.mapboxgl || !bounds) {
        throw new Error("The Mapbox terrain source is not available.");
      }

      this.showMapboxRenderer();

      if (this.mapboxRendererSuspended) {
        this.showMapboxRenderer();
        this.suspendMapStyleRestoration = false;
        this.mapboxRendererSuspended = false;
        const style = this.$map.getStyle && this.$map.getStyle();
        const styleName = String(
          (style && (style.name || style.sprite)) || "",
        ).toLowerCase();
        if (!styleName.includes("satellite")) {
          const styleLoaded = this.waitForMapEvent("style.load");
          this.$map.setStyle(MAPBOX_STYLE_SATELLITE);
          await styleLoaded;
        }
      }

      if (!this.$map.isStyleLoaded()) {
        await this.waitForMapEvent("style.load");
      }

      if (!this.$map.getSource(MAPBOX_TERRAIN_SOURCE_ID)) {
        this.$map.addSource(MAPBOX_TERRAIN_SOURCE_ID, {
          type: "raster-dem",
          url: "mapbox://mapbox.mapbox-terrain-dem-v1",
          tileSize: 512,
          maxzoom: 14,
        });
      }

      this.$map.setTerrain({
        source: MAPBOX_TERRAIN_SOURCE_ID,
        // Keep the hidden Mapbox sampler on the same terrain exaggeration as
        // Satellite. queryTerrainElevation(..., { exaggerated: false }) still
        // returns raw meters, but this prevents the sampler from leaving the
        // visible Satellite map in a flat/low-terrain state after 3D reset or
        // after switching back from the standalone Three.js view.
        exaggeration: MAPBOX_3D_TERRAIN_EXAGGERATION,
      });

      this.ensureStandaloneBuildingSource();
      this.addCustom3DBuildingsLayer();

      this.$map.fitBounds(
        [
          [Number(bounds.minLng), Number(bounds.minLat)],
          [Number(bounds.maxLng), Number(bounds.maxLat)],
        ],
        {
          padding: 56,
          duration: 0,
          maxZoom: THREE_PARK_BUILDING_QUERY_MAX_ZOOM,
          pitch: 0,
          bearing: 0,
        },
      );

      await this.waitForMapEvent("idle");
      await new Promise((resolve) =>
        window.setTimeout(resolve, THREE_PARK_BUILDING_TILE_SETTLE_MS),
      );
      await this.waitForStandaloneBuildingTiles();
    },

    waitForStandaloneBuildingTiles(timeout = THREE_PARK_IDLE_TIMEOUT_MS) {
      if (!this.$map) return Promise.resolve(false);

      return new Promise((resolve) => {
        let settled = false;
        let timer = null;

        const cleanup = () => {
          if (timer != null) window.clearTimeout(timer);
          try {
            this.$map.off("sourcedata", handleSourceData);
          } catch (error) {
            // The style/source may have been replaced while 3D was loading.
          }
        };

        const complete = (value) => {
          if (settled) return;
          settled = true;
          cleanup();
          resolve(value);
        };

        const isBuildingSourceLoaded = () => {
          try {
            return (
              this.$map.getSource(MAPBOX_3D_BUILDINGS_SOURCE_ID) &&
              typeof this.$map.isSourceLoaded === "function" &&
              this.$map.isSourceLoaded(MAPBOX_3D_BUILDINGS_SOURCE_ID)
            );
          } catch (error) {
            return false;
          }
        };

        const handleSourceData = (event) => {
          if (
            event &&
            event.sourceId &&
            event.sourceId !== MAPBOX_3D_BUILDINGS_SOURCE_ID
          ) {
            return;
          }

          if (isBuildingSourceLoaded()) complete(true);
        };

        if (isBuildingSourceLoaded()) {
          complete(true);
          return;
        }

        try {
          this.$map.on("sourcedata", handleSourceData);
        } catch (error) {
          complete(false);
          return;
        }

        timer = window.setTimeout(() => complete(false), timeout);
      });
    },

    queryStandaloneTerrainElevation(lngLat) {
      if (!this.$map || typeof this.$map.queryTerrainElevation !== "function") {
        return null;
      }

      try {
        const elevation = this.$map.queryTerrainElevation(
          {
            lng: Number(lngLat[0]),
            lat: Number(lngLat[1]),
          },
          {
            exaggerated: false,
          },
        );

        return Number.isFinite(Number(elevation)) ? Number(elevation) : null;
      } catch (error) {
        return null;
      }
    },

    getStandaloneProjectLandscapeUrls(projectItem = null) {
      const item =
        projectItem ||
        (this.currentProjectId
          ? this.getProjectItemById(this.currentProjectId)
          : null);

      if (!item || !item.landscape) return [];

      const values = Array.isArray(item.landscape)
        ? item.landscape
        : [item.landscape];

      return values
        .map((value) => String(value || "").trim())
        .filter(Boolean);
    },

    getStandaloneProjectLandscapeUrl(projectItem = null) {
      // Backward-compatible helper for code that only needs to know whether
      // landscape data exists. Multi-zone projects are represented by the
      // first URL here; terrain loading itself uses the complete URL array.
      return this.getStandaloneProjectLandscapeUrls(projectItem)[0] || "";
    },

    hasStandaloneProjectLandscapeData() {
      const afterData =
        this.threeParkModelData && this.threeParkModelData.terrainAfterData;

      // Once the 3D scene has been built, trust the actual LandXML load result.
      // This also covers a non-empty landscape column whose XML failed to load.
      if (
        afterData &&
        Object.prototype.hasOwnProperty.call(afterData, "hasLandscapeData")
      ) {
        return Boolean(afterData.hasLandscapeData);
      }

      return Boolean(this.getStandaloneProjectLandscapeUrl());
    },

    getStandaloneMissingLandscapeMessage(layerKey = "") {
      const normalizedKey = String(layerKey || "").trim();

      if (normalizedKey === "terrainAfter") {
        return this.mapUiText(
          "Terrain (after) has no ground surface because this project does not have landscape data.",
          "لا يحتوي التضاريس (بعد) على سطح أرضي لأن هذا المشروع لا يحتوي على بيانات اللاندسكيب.",
          "Terrain (after) has no ground surface because this project does not have landscape data.",
        );
      }

      return this.mapUiText(
        "This layer cannot be rendered because this project does not have landscape data.",
        "لا يمكن عرض هذه الطبقة لأن هذا المشروع لا يحتوي على بيانات اللاندسكيب.",
        "This layer cannot be rendered because this project does not have landscape data.",
      );
    },

    async loadStandaloneLandXml(url) {
      const response = await fetch(url, { cache: "no-cache" });
      if (!response.ok) {
        throw new Error(
          `Unable to load LandXML terrain (${response.status} ${response.statusText}).`,
        );
      }

      const xmlText = await response.text();
      const documentNode = new DOMParser().parseFromString(
        xmlText,
        "application/xml",
      );
      const parserError = documentNode.querySelector("parsererror");
      if (parserError) {
        throw new Error("The project's landscape file is not valid XML.");
      }

      return documentNode;
    },

    getLandXmlElements(documentNode, localName) {
      if (!documentNode || !localName) return [];

      const namespaceMatches = Array.from(
        documentNode.getElementsByTagNameNS("*", localName),
      );
      if (namespaceMatches.length) return namespaceMatches;

      return Array.from(documentNode.getElementsByTagName(localName));
    },

    buildLandXmlBoundaryLoops(pointById, faces) {
      const edgeCounts = new Map();
      const addEdge = (startId, endId) => {
        const a = String(startId);
        const b = String(endId);
        const key = a < b ? `${a}|${b}` : `${b}|${a}`;
        const existing = edgeCounts.get(key);
        if (existing) {
          existing.count += 1;
        } else {
          edgeCounts.set(key, { count: 1, a, b });
        }
      };

      faces.forEach((face) => {
        addEdge(face[0], face[1]);
        addEdge(face[1], face[2]);
        addEdge(face[2], face[0]);
      });

      const adjacency = new Map();
      const unusedEdges = new Set();
      edgeCounts.forEach((edge, key) => {
        if (edge.count !== 1) return;
        unusedEdges.add(key);
        if (!adjacency.has(edge.a)) adjacency.set(edge.a, []);
        if (!adjacency.has(edge.b)) adjacency.set(edge.b, []);
        adjacency.get(edge.a).push(edge.b);
        adjacency.get(edge.b).push(edge.a);
      });

      const edgeKey = (a, b) =>
        String(a) < String(b) ? `${a}|${b}` : `${b}|${a}`;
      const loops = [];

      while (unusedEdges.size) {
        const firstKey = unusedEdges.values().next().value;
        const [startId, firstNextId] = firstKey.split("|");
        const loopIds = [startId];
        let previousId = startId;
        let currentId = firstNextId;
        unusedEdges.delete(firstKey);

        let guard = 0;
        while (guard < edgeCounts.size + 5) {
          guard += 1;
          loopIds.push(currentId);
          if (currentId === startId) break;

          const candidates = (adjacency.get(currentId) || []).filter((nextId) =>
            unusedEdges.has(edgeKey(currentId, nextId)),
          );
          if (!candidates.length) break;

          const nextId =
            candidates.find((candidate) => candidate !== previousId) ||
            candidates[0];
          unusedEdges.delete(edgeKey(currentId, nextId));
          previousId = currentId;
          currentId = nextId;
        }

        if (loopIds.length > 3 && loopIds[loopIds.length - 1] === startId) {
          loopIds.pop();
          const ring = loopIds
            .map((id) => pointById.get(String(id)))
            .filter(Boolean)
            .map((point) => ({ x: point.x, z: point.z }));
          if (ring.length >= 3) loops.push(ring);
        }
      }

      return loops;
    },

    getTerrainRingSignedArea(ring = []) {
      let area = 0;
      for (let index = 0; index < ring.length; index += 1) {
        const current = ring[index];
        const next = ring[(index + 1) % ring.length];
        area += Number(current.x) * Number(next.z);
        area -= Number(next.x) * Number(current.z);
      }
      return area / 2;
    },

    async buildStandaloneTerrainDataFromLandXml(
      landscapeUrl,
      origin,
      bounds,
      buildToken,
    ) {
      const landscapeUrls = Array.isArray(landscapeUrl)
        ? landscapeUrl.filter(Boolean)
        : landscapeUrl
          ? [landscapeUrl]
          : [];
      if (!landscapeUrls.length) {
        throw new Error("No LandXML terrain files were provided.");
      }

      // A project may use either:
      //   1) several LandXML files (one surface/zone per file), or
      //   2) one combined LandXML file containing several <Surface> elements.
      // Treat every <Surface> independently so Civil 3D point IDs may safely
      // restart at 1 inside each surface and each zone receives its own boundary.
      const rawPoints = [];
      const faces = [];
      const zoneFaceGroups = [];
      const loadedLandscapeUrls = [];
      const loadedLandscapeSurfaces = [];
      const failedLandscapeZones = [];
      const failedLandscapeSurfaces = [];

      for (let fileIndex = 0; fileIndex < landscapeUrls.length; fileIndex += 1) {
        const zoneUrl = landscapeUrls[fileIndex];
        let fileLoadedSurfaceCount = 0;

        try {
          const documentNode = await this.loadStandaloneLandXml(zoneUrl);
          if (buildToken !== this.threeParkBuildToken) {
            throw new Error("The 3D terrain build was cancelled.");
          }

          const surfaceElements = this.getLandXmlElements(
            documentNode,
            "Surface",
          );

          // Some older/simpler exports may omit the <Surface> wrapper. Keep a
          // document-level fallback for backward compatibility with those files.
          const surfacesToProcess = surfaceElements.length
            ? surfaceElements
            : [documentNode];

          for (
            let surfaceIndex = 0;
            surfaceIndex < surfacesToProcess.length;
            surfaceIndex += 1
          ) {
            const surfaceNode = surfacesToProcess[surfaceIndex];
            const surfaceName =
              (surfaceNode &&
                typeof surfaceNode.getAttribute === "function" &&
                String(surfaceNode.getAttribute("name") || "").trim()) ||
              `Surface ${surfaceIndex + 1}`;

            try {
              const pointElements = this.getLandXmlElements(surfaceNode, "P");
              const faceElements = this.getLandXmlElements(surfaceNode, "F");
              if (!pointElements.length || !faceElements.length) {
                throw new Error("This surface does not contain a TIN.");
              }

              // Namespace IDs by BOTH file and surface. Combined LandXML files
              // commonly restart point IDs at 1 for every <Surface>.
              const idPrefix = `${fileIndex}:${surfaceIndex}:`;
              const surfacePointIds = new Set();
              const surfacePoints = [];
              const surfaceFaces = [];

              pointElements.forEach((element) => {
                const values = String(element.textContent || "")
                  .trim()
                  .split(/\s+/)
                  .map(Number);
                const sourceId = String(
                  element.getAttribute("id") || "",
                ).trim();
                if (
                  values.length < 3 ||
                  !sourceId ||
                  !values.slice(0, 3).every(Number.isFinite)
                ) {
                  return;
                }

                surfacePointIds.add(sourceId);
                surfacePoints.push({
                  id: `${idPrefix}${sourceId}`,
                  northing: values[0],
                  easting: values[1],
                  elevation: values[2],
                });
              });

              faceElements.forEach((element) => {
                const sourceFace = String(element.textContent || "")
                  .trim()
                  .split(/\s+/)
                  .slice(0, 3);
                if (
                  sourceFace.length === 3 &&
                  sourceFace.every((pointId) =>
                    surfacePointIds.has(pointId),
                  )
                ) {
                  surfaceFaces.push(
                    sourceFace.map(
                      (pointId) => `${idPrefix}${pointId}`,
                    ),
                  );
                }
              });

              if (!surfacePoints.length || !surfaceFaces.length) {
                throw new Error(
                  "This surface has no usable TIN vertices/faces.",
                );
              }

              // Avoid push(...hugeArray): large LandXML TINs can exceed the JS
              // function-argument limit and throw Maximum call stack size exceeded.
              for (
                let pointIndex = 0;
                pointIndex < surfacePoints.length;
                pointIndex += 1
              ) {
                rawPoints.push(surfacePoints[pointIndex]);
              }
              for (
                let faceIndex = 0;
                faceIndex < surfaceFaces.length;
                faceIndex += 1
              ) {
                faces.push(surfaceFaces[faceIndex]);
              }

              zoneFaceGroups.push({
                fileIndex,
                surfaceIndex,
                zoneUrl,
                surfaceName,
                faces: surfaceFaces,
              });
              loadedLandscapeSurfaces.push({
                fileIndex,
                surfaceIndex,
                url: zoneUrl,
                name: surfaceName,
                pointCount: surfacePoints.length,
                faceCount: surfaceFaces.length,
              });
              fileLoadedSurfaceCount += 1;

              console.info(
                `Loaded LandXML surface ${surfaceIndex + 1}/${surfacesToProcess.length} from file ${fileIndex + 1}/${landscapeUrls.length}: ${surfaceName} (${surfacePoints.length} points, ${surfaceFaces.length} faces)`,
              );
            } catch (surfaceError) {
              if (buildToken !== this.threeParkBuildToken) throw surfaceError;

              const message =
                surfaceError && surfaceError.message
                  ? surfaceError.message
                  : String(surfaceError);
              failedLandscapeSurfaces.push({
                fileIndex,
                surfaceIndex,
                url: zoneUrl,
                name: surfaceName,
                message,
              });
              console.warn(
                `Skipping LandXML surface ${surfaceIndex + 1}/${surfacesToProcess.length} from ${zoneUrl}: ${surfaceName}`,
                surfaceError,
              );
            }

            // Yield between surfaces in a combined file. The simplified KAP
            // landscape contains four surfaces in one XML, and this avoids one
            // uninterrupted CPU spike while the TIN arrays are being created.
            await new Promise((resolve) => setTimeout(resolve, 0));
          }

          if (!fileLoadedSurfaceCount) {
            throw new Error(
              "This file does not contain any usable TIN surfaces.",
            );
          }

          loadedLandscapeUrls.push(zoneUrl);
          console.info(
            `Loaded LandXML file ${fileIndex + 1}/${landscapeUrls.length}: ${zoneUrl} (${fileLoadedSurfaceCount} usable surface${fileLoadedSurfaceCount === 1 ? "" : "s"})`,
          );
        } catch (error) {
          // Cancellation is intentional and must still abort the whole build.
          if (buildToken !== this.threeParkBuildToken) throw error;

          const message = error && error.message ? error.message : String(error);
          failedLandscapeZones.push({
            fileIndex,
            zoneNumber: fileIndex + 1,
            url: zoneUrl,
            message,
          });
          console.warn(
            `Skipping LandXML file ${fileIndex + 1}/${landscapeUrls.length}: ${zoneUrl}`,
            error,
          );
        }

        // Strictly sequential file loading. Give the browser a rendering/GC
        // opportunity before fetching/parsing the next XML file.
        await new Promise((resolve) => setTimeout(resolve, 16));
      }

      if (!rawPoints.length || !faces.length || !loadedLandscapeUrls.length) {
        const surfaceDetails = failedLandscapeSurfaces
          .map(
            (surface) =>
              `${surface.name || `Surface ${surface.surfaceIndex + 1}`}: ${surface.message}`,
          )
          .join(" | ");
        const fileDetails = failedLandscapeZones
          .map((zone) => `File ${zone.zoneNumber}: ${zone.message}`)
          .join(" | ");
        const details = [surfaceDetails, fileDetails].filter(Boolean).join(" | ");
        throw new Error(
          details
            ? `None of the LandXML surfaces could be used. ${details}`
            : "The LandXML TIN vertices or faces are invalid.",
        );
      }

      const sourceBounds = rawPoints.reduce(
        (output, point) => {
          output.minEasting = Math.min(output.minEasting, point.easting);
          output.maxEasting = Math.max(output.maxEasting, point.easting);
          output.minNorthing = Math.min(output.minNorthing, point.northing);
          output.maxNorthing = Math.max(output.maxNorthing, point.northing);
          return output;
        },
        {
          minEasting: Infinity,
          maxEasting: -Infinity,
          minNorthing: Infinity,
          maxNorthing: -Infinity,
        },
      );
      const sourceCenterEasting =
        (sourceBounds.minEasting + sourceBounds.maxEasting) / 2;
      const sourceCenterNorthing =
        (sourceBounds.minNorthing + sourceBounds.maxNorthing) / 2;

      const pointById = new Map();
      const pointMap = new Map();
      rawPoints.forEach((rawPoint) => {
        const point = {
          id: rawPoint.id,
          x: rawPoint.easting - sourceCenterEasting,
          z: -(rawPoint.northing - sourceCenterNorthing),
          elevation: rawPoint.elevation,
        };
        pointById.set(rawPoint.id, point);
        pointMap.set(this.getTerrainPointKey(point), point);
      });

      // Keep only the triangle vertex keys for the rendered LandXML surface.
      // Do not create a second full triangle copy for the island bottom.
      const baseTriangles = [];
      const triangleVertexKeys = [];
      faces.forEach((face) => {
        face.forEach((pointId) => {
          const point = pointById.get(pointId);
          if (point) {
            triangleVertexKeys.push(this.getTerrainPointKey(point));
          }
        });
      });

      // Every LandXML <Surface> is an independent terrain part. This is
      // essential for a single combined KAP XML: Zone B/C/D must not be treated
      // as holes inside Zone A just because all four surfaces share one file.
      const localParts = [];
      zoneFaceGroups.forEach((zoneGroup) => {
        const boundaryLoops = this.buildLandXmlBoundaryLoops(
          pointById,
          zoneGroup.faces,
        );
        boundaryLoops.sort(
          (a, b) =>
            Math.abs(this.getTerrainRingSignedArea(b)) -
            Math.abs(this.getTerrainRingSignedArea(a)),
        );
        if (boundaryLoops.length) {
          localParts.push({
            outer: boundaryLoops[0],
            holes: boundaryLoops.slice(1),
            surfaceName: zoneGroup.surfaceName,
          });
        }
      });

      if (!localParts.length) {
        localParts.push({
          outer: [
            {
              x: sourceBounds.minEasting - sourceCenterEasting,
              z: -(sourceBounds.minNorthing - sourceCenterNorthing),
            },
            {
              x: sourceBounds.maxEasting - sourceCenterEasting,
              z: -(sourceBounds.minNorthing - sourceCenterNorthing),
            },
            {
              x: sourceBounds.maxEasting - sourceCenterEasting,
              z: -(sourceBounds.maxNorthing - sourceCenterNorthing),
            },
            {
              x: sourceBounds.minEasting - sourceCenterEasting,
              z: -(sourceBounds.maxNorthing - sourceCenterNorthing),
            },
          ],
          holes: [],
        });
      }

      // Never spread a large elevations array into Math.min/Math.max.
      let minimumElevation = Infinity;
      let maximumElevation = -Infinity;
      rawPoints.forEach((point) => {
        const elevation = Number(point.elevation);
        if (!Number.isFinite(elevation)) return;
        if (elevation < minimumElevation) minimumElevation = elevation;
        if (elevation > maximumElevation) maximumElevation = elevation;
      });

      if (
        !Number.isFinite(minimumElevation) ||
        !Number.isFinite(maximumElevation)
      ) {
        throw new Error("The LandXML terrain elevations are invalid.");
      }

      const rawElevationRange = Math.max(
        0,
        maximumElevation - minimumElevation,
      );
      const verticalExaggeration =
        rawElevationRange > 0.05
          ? Math.max(
              THREE_PARK_MIN_VERTICAL_EXAGGERATION,
              Math.min(
                THREE_PARK_MAX_VERTICAL_EXAGGERATION,
                MAPBOX_3D_TERRAIN_EXAGGERATION,
              ),
            )
          : 1;
      const maxDimension = Math.max(
        sourceBounds.maxEasting - sourceBounds.minEasting,
        sourceBounds.maxNorthing - sourceBounds.minNorthing,
        1,
      );
      const islandWallDepth = Math.max(
        THREE_PARK_MIN_ISLAND_WALL_DEPTH_METERS,
        Math.min(
          THREE_PARK_MAX_ISLAND_WALL_DEPTH_METERS,
          maxDimension * THREE_PARK_ISLAND_WALL_DEPTH_RATIO,
        ),
      );
      const boundarySampleSpacing = Math.max(
        THREE_PARK_MIN_BOUNDARY_SAMPLE_SPACING_METERS,
        Math.min(
          THREE_PARK_MAX_BOUNDARY_SAMPLE_SPACING_METERS,
          maxDimension / THREE_PARK_BOUNDARY_SAMPLE_DIVISIONS,
        ),
      );
      const gridSpacing = Math.max(
        THREE_PARK_MIN_GRID_SPACING,
        Math.min(
          THREE_PARK_MAX_GRID_SPACING,
          maxDimension / THREE_PARK_GRID_DIVISIONS,
        ),
      );
      const localBounds = {
        minX: sourceBounds.minEasting - sourceCenterEasting,
        maxX: sourceBounds.maxEasting - sourceCenterEasting,
        minZ: -(sourceBounds.maxNorthing - sourceCenterNorthing),
        maxZ: -(sourceBounds.minNorthing - sourceCenterNorthing),
      };

      await this.ensureStandaloneBuildingTilesForPark(
        bounds,
        origin,
        localParts,
      );
      const buildingFootprints = this.queryStandaloneBuildingFootprints(
        bounds,
        origin,
        localParts,
        pointMap,
        minimumElevation,
        verticalExaggeration,
      );

      return {
        localParts,
        baseTriangles,
        triangleVertexKeys,
        pointMap,
        minimumElevation,
        maximumElevation,
        rawElevationRange,
        verticalExaggeration,
        visibleElevationRange: rawElevationRange * verticalExaggeration,
        maxDimension,
        islandWallDepth,
        boundarySampleSpacing,
        gridSpacing,
        localBounds,
        geographicBounds: { ...bounds },
        origin: origin.slice(),
        satelliteTexture: null,
        buildingFootprints,
        hasTerrainData: true,
        terrainSource: "landxml",
        landscapeUrl:
          loadedLandscapeUrls.length === 1
            ? loadedLandscapeUrls[0]
            : loadedLandscapeUrls,
        configuredLandscapeUrls: landscapeUrls.slice(),
        loadedLandscapeUrls: loadedLandscapeUrls.slice(),
        loadedLandscapeSurfaces: loadedLandscapeSurfaces.slice(),
        failedLandscapeZones: failedLandscapeZones.slice(),
        failedLandscapeSurfaces: failedLandscapeSurfaces.slice(),
        hasPartialLandscapeData:
          failedLandscapeZones.length > 0 ||
          failedLandscapeSurfaces.length > 0,
      };
    },

    async buildStandaloneTerrainData(
      parts,
      origin,
      bounds,
      buildToken,
      useLandscape = true,
    ) {
      const landscapeUrls = useLandscape
        ? this.getStandaloneProjectLandscapeUrls()
        : [];
      if (landscapeUrls.length) {
        try {
          return await this.buildStandaloneTerrainDataFromLandXml(
            landscapeUrls,
            origin,
            bounds,
            buildToken,
          );
        } catch (error) {
          console.warn(
            "LandXML terrain could not be used; falling back to Mapbox terrain.",
            error,
          );
        }
      }

      const localParts = [];
      const baseTriangles = [];
      let faceCount = 0;

      parts.forEach((part) => {
        const outer = part.outer.map((coordinate) =>
          this.lngLatToTerrainMeters(coordinate, origin),
        );
        const holes = (part.holes || []).map((ring) =>
          ring.map((coordinate) =>
            this.lngLatToTerrainMeters(coordinate, origin),
          ),
        );

        let contour = outer.map((point) => new THREE.Vector2(point.x, point.z));
        const holeVectors = holes.map((ring) =>
          ring.map((point) => new THREE.Vector2(point.x, point.z)),
        );
        let faces = THREE.ShapeUtils.triangulateShape(contour, holeVectors);

        if (!faces.length) {
          contour = contour.slice().reverse();
          faces = THREE.ShapeUtils.triangulateShape(contour, holeVectors);
        }

        const flattened = contour.concat(...holeVectors);

        faces.forEach((face) => {
          const triangle = face.map((index) => ({
            x: flattened[index].x,
            z: flattened[index].y,
          }));
          if (triangle.length === 3) {
            baseTriangles.push(triangle);
            faceCount += 1;
          }
        });

        localParts.push({ outer, holes });
      });

      if (!baseTriangles.length) {
        throw new Error("The project geometry could not be triangulated.");
      }

      const widthMeters =
        (Number(bounds.maxLng) - Number(bounds.minLng)) *
        111320 *
        Math.max(
          0.15,
          Math.cos(
            (((Number(bounds.minLat) + Number(bounds.maxLat)) / 2) * Math.PI) /
              180,
          ),
        );
      const depthMeters =
        (Number(bounds.maxLat) - Number(bounds.minLat)) * 111320;
      const maxDimension = Math.max(widthMeters, depthMeters, 1);
      const subdivisions = this.getTerrainSubdivisionLevel(
        faceCount,
        maxDimension,
      );
      const boundarySampleSpacing = Math.max(
        THREE_PARK_MIN_BOUNDARY_SAMPLE_SPACING_METERS,
        Math.min(
          THREE_PARK_MAX_BOUNDARY_SAMPLE_SPACING_METERS,
          maxDimension / THREE_PARK_BOUNDARY_SAMPLE_DIVISIONS,
        ),
      );
      const sampledLocalParts = localParts.map((part) => ({
        outer: this.densifyTerrainRing(part.outer, boundarySampleSpacing),
        holes: (part.holes || []).map((ring) =>
          this.densifyTerrainRing(ring, boundarySampleSpacing),
        ),
      }));

      const uniquePoints = new Map();
      const triangleVertexKeys = [];

      const registerPoint = (point) => {
        const key = this.getTerrainPointKey(point);
        if (!uniquePoints.has(key)) {
          uniquePoints.set(key, {
            key,
            x: Number(point.x),
            z: Number(point.z),
            elevation: null,
          });
        }
        return key;
      };

      const pushTriangle = (a, b, c) => {
        triangleVertexKeys.push(
          registerPoint(a),
          registerPoint(b),
          registerPoint(c),
        );
      };

      baseTriangles.forEach(([a, b, c]) => {
        for (let row = 0; row < subdivisions; row += 1) {
          for (let column = 0; column < subdivisions - row; column += 1) {
            const u = row / subdivisions;
            const v = column / subdivisions;
            const nextU = (row + 1) / subdivisions;
            const nextV = (column + 1) / subdivisions;

            const p0 = this.getBarycentricTerrainPoint(a, b, c, u, v);
            const p1 = this.getBarycentricTerrainPoint(a, b, c, nextU, v);
            const p2 = this.getBarycentricTerrainPoint(a, b, c, u, nextV);

            pushTriangle(p0, p1, p2);

            if (column < subdivisions - row - 1) {
              const p3 = this.getBarycentricTerrainPoint(a, b, c, nextU, nextV);
              pushTriangle(p1, p3, p2);
            }
          }
        }
      });

      sampledLocalParts.forEach((part) => {
        [part.outer, ...(part.holes || [])].forEach((ring) => {
          ring.forEach(registerPoint);
        });
      });

      const points = Array.from(uniquePoints.values());
      const validElevations = [];

      for (let index = 0; index < points.length; index += 1) {
        if (buildToken !== this.threeParkBuildToken) {
          throw new Error("The 3D terrain build was cancelled.");
        }

        const point = points[index];
        const lngLat = this.terrainMetersToLngLat(point, origin);
        const elevation = this.queryStandaloneTerrainElevation(lngLat);
        point.elevation = elevation;

        if (Number.isFinite(Number(elevation))) {
          validElevations.push(Number(elevation));
        }

        if (index > 0 && index % 250 === 0) {
          await new Promise((resolve) => window.requestAnimationFrame(resolve));
        }
      }

      const fallbackElevation = validElevations.length
        ? validElevations.reduce((sum, value) => sum + value, 0) /
          validElevations.length
        : 0;

      points.forEach((point) => {
        if (!Number.isFinite(Number(point.elevation))) {
          point.elevation = fallbackElevation;
        }
      });

      const elevations = points.map((point) => Number(point.elevation));
      const minimumElevation = Math.min(...elevations);
      const maximumElevation = Math.max(...elevations);
      const rawElevationRange = Math.max(
        0,
        Number(maximumElevation) - Number(minimumElevation),
      );
      const verticalExaggeration =
        validElevations.length > 0 && rawElevationRange > 0.05
          ? Math.max(
              THREE_PARK_MIN_VERTICAL_EXAGGERATION,
              Math.min(
                THREE_PARK_MAX_VERTICAL_EXAGGERATION,
                MAPBOX_3D_TERRAIN_EXAGGERATION,
              ),
            )
          : 1;
      const islandWallDepth = Math.max(
        THREE_PARK_MIN_ISLAND_WALL_DEPTH_METERS,
        Math.min(
          THREE_PARK_MAX_ISLAND_WALL_DEPTH_METERS,
          maxDimension * THREE_PARK_ISLAND_WALL_DEPTH_RATIO,
        ),
      );
      const gridSpacing = Math.max(
        THREE_PARK_MIN_GRID_SPACING,
        Math.min(
          THREE_PARK_MAX_GRID_SPACING,
          maxDimension / THREE_PARK_GRID_DIVISIONS,
        ),
      );

      const pointXs = points.map((point) => Number(point.x));
      const pointZs = points.map((point) => Number(point.z));
      const localBounds = {
        minX: Math.min(...pointXs),
        maxX: Math.max(...pointXs),
        minZ: Math.min(...pointZs),
        maxZ: Math.max(...pointZs),
      };

      await this.ensureStandaloneBuildingTilesForPark(
        bounds,
        origin,
        sampledLocalParts,
      );

      const buildingFootprints = this.queryStandaloneBuildingFootprints(
        bounds,
        origin,
        sampledLocalParts,
        uniquePoints,
        minimumElevation,
        verticalExaggeration,
      );

      return {
        localParts: sampledLocalParts,
        baseTriangles,
        triangleVertexKeys,
        pointMap: uniquePoints,
        minimumElevation,
        maximumElevation,
        rawElevationRange,
        verticalExaggeration,
        visibleElevationRange: rawElevationRange * verticalExaggeration,
        maxDimension,
        islandWallDepth,
        boundarySampleSpacing,
        gridSpacing,
        localBounds,
        geographicBounds: { ...bounds },
        origin: origin.slice(),
        satelliteTexture: null,
        buildingFootprints,
        hasTerrainData: validElevations.length > 0,
        terrainSource: "mapbox",
        landscapeUrl: "",
      };
    },

    isTerrainPointInsideRing(point, ring = []) {
      if (!point || !Array.isArray(ring) || ring.length < 3) return false;

      let inside = false;
      const x = Number(point.x);
      const z = Number(point.z);

      for (let i = 0, j = ring.length - 1; i < ring.length; j = i++) {
        const xi = Number(ring[i].x);
        const zi = Number(ring[i].z);
        const xj = Number(ring[j].x);
        const zj = Number(ring[j].z);
        const intersects =
          zi > z !== zj > z &&
          x < ((xj - xi) * (z - zi)) / Math.max(1e-9, zj - zi) + xi;
        if (intersects) inside = !inside;
      }

      return inside;
    },

    isTerrainPointInsidePart(point, part) {
      if (!part || !this.isTerrainPointInsideRing(point, part.outer)) {
        return false;
      }

      return !(part.holes || []).some((hole) =>
        this.isTerrainPointInsideRing(point, hole),
      );
    },

    isLngLatInsideTerrainParts(coordinate, origin, localParts = []) {
      if (!Array.isArray(coordinate) || coordinate.length < 2 || !origin) {
        return false;
      }

      const point = this.lngLatToTerrainMeters(coordinate, origin);
      return localParts.some((part) =>
        this.isTerrainPointInsidePart(point, part),
      );
    },

    isTerrainPointInsideAnyPart(point, localParts = []) {
      if (!point || !Array.isArray(localParts) || !localParts.length) {
        return false;
      }

      return localParts.some((part) =>
        this.isTerrainPointInsidePart(point, part),
      );
    },

    isTerrainPointSafelyInsideAnyPart(
      point,
      localParts = [],
      clearanceMeters = 0,
    ) {
      if (!this.isTerrainPointInsideAnyPart(point, localParts)) return false;

      const clearance = Math.max(0, Number(clearanceMeters) || 0);
      if (clearance <= 0.001) return true;

      // A point can technically be inside the project polygon while most of a
      // tree crown or pole base hangs beyond the clipped floating-island edge.
      // Require a small circular footprint around the object to remain inside.
      const samples = 12;
      for (let index = 0; index < samples; index += 1) {
        const angle = (index / samples) * Math.PI * 2;
        const sample = {
          x: Number(point.x) + Math.cos(angle) * clearance,
          z: Number(point.z) + Math.sin(angle) * clearance,
        };
        if (!this.isTerrainPointInsideAnyPart(sample, localParts)) {
          return false;
        }
      }

      return true;
    },

    getTerrainRingInsideRatio(localRing = [], localParts = []) {
      if (
        !Array.isArray(localRing) ||
        !localRing.length ||
        !Array.isArray(localParts) ||
        !localParts.length
      ) {
        return 0;
      }

      let tested = 0;
      let inside = 0;

      localRing.forEach((point) => {
        if (
          !point ||
          !Number.isFinite(Number(point.x)) ||
          !Number.isFinite(Number(point.z))
        ) {
          return;
        }

        tested += 1;
        if (this.isTerrainPointInsideAnyPart(point, localParts)) inside += 1;
      });

      return tested > 0 ? inside / tested : 0;
    },

    getTerrainSegmentOrientation(a, b, c) {
      return (
        (Number(b.z) - Number(a.z)) * (Number(c.x) - Number(b.x)) -
        (Number(b.x) - Number(a.x)) * (Number(c.z) - Number(b.z))
      );
    },

    isTerrainPointOnSegment(point, a, b, tolerance = 1e-6) {
      if (!point || !a || !b) return false;

      const cross =
        (Number(point.z) - Number(a.z)) * (Number(b.x) - Number(a.x)) -
        (Number(point.x) - Number(a.x)) * (Number(b.z) - Number(a.z));
      if (Math.abs(cross) > tolerance) return false;

      return (
        Number(point.x) <= Math.max(Number(a.x), Number(b.x)) + tolerance &&
        Number(point.x) >= Math.min(Number(a.x), Number(b.x)) - tolerance &&
        Number(point.z) <= Math.max(Number(a.z), Number(b.z)) + tolerance &&
        Number(point.z) >= Math.min(Number(a.z), Number(b.z)) - tolerance
      );
    },

    doTerrainSegmentsIntersect(a, b, c, d) {
      if (!a || !b || !c || !d) return false;

      const o1 = this.getTerrainSegmentOrientation(a, b, c);
      const o2 = this.getTerrainSegmentOrientation(a, b, d);
      const o3 = this.getTerrainSegmentOrientation(c, d, a);
      const o4 = this.getTerrainSegmentOrientation(c, d, b);
      const tolerance = 1e-6;

      if (Math.abs(o1) <= tolerance && this.isTerrainPointOnSegment(c, a, b))
        return true;
      if (Math.abs(o2) <= tolerance && this.isTerrainPointOnSegment(d, a, b))
        return true;
      if (Math.abs(o3) <= tolerance && this.isTerrainPointOnSegment(a, c, d))
        return true;
      if (Math.abs(o4) <= tolerance && this.isTerrainPointOnSegment(b, c, d))
        return true;

      return o1 > 0 !== o2 > 0 && o3 > 0 !== o4 > 0;
    },

    getTerrainRingEdges(ring = []) {
      const points = (Array.isArray(ring) ? ring : []).filter(
        (point) =>
          point &&
          Number.isFinite(Number(point.x)) &&
          Number.isFinite(Number(point.z)),
      );
      if (points.length < 2) return [];

      return points.map((point, index) => [
        point,
        points[(index + 1) % points.length],
      ]);
    },

    doTerrainRingsHaveIntersectingEdges(ringA = [], ringB = []) {
      const edgesA = this.getTerrainRingEdges(ringA);
      const edgesB = this.getTerrainRingEdges(ringB);
      if (!edgesA.length || !edgesB.length) return false;

      return edgesA.some(([a, b]) =>
        edgesB.some(([c, d]) => this.doTerrainSegmentsIntersect(a, b, c, d)),
      );
    },

    getStandaloneBuildingParkAnchor(
      localOuterRing = [],
      localCentroid,
      localParts = [],
    ) {
      if (
        !Array.isArray(localOuterRing) ||
        localOuterRing.length < 3 ||
        !Array.isArray(localParts) ||
        !localParts.length
      ) {
        return null;
      }

      if (
        localCentroid &&
        this.isTerrainPointInsideAnyPart(localCentroid, localParts)
      ) {
        return localCentroid;
      }

      const insideVertex = localOuterRing.find((point) =>
        this.isTerrainPointInsideAnyPart(point, localParts),
      );
      if (insideVertex) return insideVertex;

      const insideEdgeMidpoint = this.getTerrainRingEdges(localOuterRing)
        .map(([a, b]) => ({
          x: (Number(a.x) + Number(b.x)) / 2,
          z: (Number(a.z) + Number(b.z)) / 2,
        }))
        .find((point) => this.isTerrainPointInsideAnyPart(point, localParts));
      if (insideEdgeMidpoint) return insideEdgeMidpoint;

      const buildingRing = localOuterRing;
      for (const part of localParts) {
        const parkRings = [part.outer, ...(part.holes || [])];
        const parkVertexInsideBuilding = (part.outer || []).find((point) =>
          this.isTerrainPointInsideRing(point, buildingRing),
        );
        if (parkVertexInsideBuilding) return parkVertexInsideBuilding;

        const hasBoundaryIntersection = parkRings.some((ring) =>
          this.doTerrainRingsHaveIntersectingEdges(buildingRing, ring),
        );
        if (hasBoundaryIntersection) {
          for (const [buildingA, buildingB] of this.getTerrainRingEdges(
            buildingRing,
          )) {
            const midpoint = {
              x: (Number(buildingA.x) + Number(buildingB.x)) / 2,
              z: (Number(buildingA.z) + Number(buildingB.z)) / 2,
            };
            if (this.isTerrainPointInsideAnyPart(midpoint, localParts)) {
              return midpoint;
            }
          }
          if (
            localCentroid &&
            this.isTerrainPointInsideRing(localCentroid, part.outer)
          ) {
            return localCentroid;
          }
        }
      }

      return null;
    },

    doesStandaloneBuildingBelongToPark(
      localOuterRing = [],
      localCentroid,
      localParts = [],
    ) {
      if (!Array.isArray(localParts) || !localParts.length) return false;

      const centroidInside =
        localCentroid &&
        this.isTerrainPointInsideAnyPart(localCentroid, localParts);
      const insideRatio = this.getTerrainRingInsideRatio(
        localOuterRing,
        localParts,
      );
      const footprintFullyInside = (
        Array.isArray(localOuterRing) ? localOuterRing : []
      ).every((point) => this.isTerrainPointInsideAnyPart(point, localParts));

      // Final rule: 3D view must never show Mapbox buildings outside the park.
      // Therefore we only render buildings whose footprint is fully inside the
      // current project GIS polygon. If a Mapbox tile returns a clipped or
      // boundary-crossing footprint, skip it instead of shrinking/snapping it to
      // the island edge.
      return Boolean(
        centroidInside &&
        footprintFullyInside &&
        insideRatio >= THREE_PARK_BUILDING_MIN_INSIDE_RATIO,
      );
    },
    getLngLatOffsetCoordinate(coordinate, eastMeters = 0, northMeters = 0) {
      const lng = Number(coordinate && coordinate[0]);
      const lat = Number(coordinate && coordinate[1]);
      if (!Number.isFinite(lng) || !Number.isFinite(lat)) return null;

      const latitudeScale = 111320;
      const longitudeScale =
        latitudeScale * Math.max(0.15, Math.cos((lat * Math.PI) / 180));

      return [
        lng + Number(eastMeters || 0) / longitudeScale,
        lat + Number(northMeters || 0) / latitudeScale,
      ];
    },

    createPointBuildingFootprint(
      coordinate,
      sizeMeters = MAPBOX_BUILDING_POINT_FOOTPRINT_METERS,
    ) {
      const half =
        Math.max(
          3,
          Number(sizeMeters) || MAPBOX_BUILDING_POINT_FOOTPRINT_METERS,
        ) / 2;
      const corners = [
        this.getLngLatOffsetCoordinate(coordinate, -half, -half),
        this.getLngLatOffsetCoordinate(coordinate, half, -half),
        this.getLngLatOffsetCoordinate(coordinate, half, half),
        this.getLngLatOffsetCoordinate(coordinate, -half, half),
      ].filter(Boolean);

      return corners.length === 4 ? [[...corners, corners[0]]] : null;
    },

    getFeatureCoordinateRings(featureOrGeometry) {
      const geometry =
        featureOrGeometry && featureOrGeometry.geometry
          ? featureOrGeometry.geometry
          : featureOrGeometry;
      const properties =
        featureOrGeometry && featureOrGeometry.properties
          ? featureOrGeometry.properties
          : {};
      if (!geometry || !Array.isArray(geometry.coordinates)) return [];

      if (geometry.type === "Polygon") {
        return [geometry.coordinates];
      }

      if (geometry.type === "MultiPolygon") {
        return geometry.coordinates;
      }

      // Some Mapbox Standard 3D objects / landmarks can be returned as points
      // instead of polygon footprints. Satellite can still draw them as 3D
      // objects, so approximate them as small footprint boxes and then run the
      // same strict inside-park filter before adding them to the island.
      if (geometry.type === "Point") {
        const height = this.getBuildingHeightFromProperties(properties);
        const size = Math.max(
          8,
          Math.min(32, Math.sqrt(Math.max(16, height * 5))),
        );
        const footprint = this.createPointBuildingFootprint(
          geometry.coordinates,
          size,
        );
        return footprint ? [footprint] : [];
      }

      if (geometry.type === "MultiPoint") {
        return geometry.coordinates
          .map((coordinate) => {
            const footprint = this.createPointBuildingFootprint(coordinate);
            return footprint;
          })
          .filter(Boolean);
      }

      return [];
    },

    getRingCentroid(coordinates = []) {
      const valid = (Array.isArray(coordinates) ? coordinates : [])
        .map((coordinate) => [
          Number(coordinate && coordinate[0]),
          Number(coordinate && coordinate[1]),
        ])
        .filter((coordinate) => coordinate.every(Number.isFinite));

      if (!valid.length) return null;

      const sum = valid.reduce(
        (accumulator, coordinate) => {
          accumulator[0] += coordinate[0];
          accumulator[1] += coordinate[1];
          return accumulator;
        },
        [0, 0],
      );

      return [sum[0] / valid.length, sum[1] / valid.length];
    },

    getBuildingHeightFromProperties(properties = {}) {
      const readNumber = (...keys) => {
        for (const key of keys) {
          const value = Number(properties && properties[key]);
          if (Number.isFinite(value) && value > 0) return value;
        }
        return null;
      };

      const explicitHeight = readNumber("height", "render_height");
      if (explicitHeight) return explicitHeight;

      const levels = readNumber("levels", "building:levels");
      if (levels) return levels * 3.5;

      return 10;
    },

    getBuildingMinHeightFromProperties(properties = {}) {
      const minHeight = Number(
        properties.min_height ?? properties.render_min_height ?? 0,
      );
      return Number.isFinite(minHeight) && minHeight > 0 ? minHeight : 0;
    },
    getLocalRingBounds(localRing = []) {
      const valid = (Array.isArray(localRing) ? localRing : []).filter(
        (point) =>
          point &&
          Number.isFinite(Number(point.x)) &&
          Number.isFinite(Number(point.z)),
      );
      if (!valid.length) return null;

      const xs = valid.map((point) => Number(point.x));
      const zs = valid.map((point) => Number(point.z));
      return {
        minX: Math.min(...xs),
        maxX: Math.max(...xs),
        minZ: Math.min(...zs),
        maxZ: Math.max(...zs),
      };
    },

    getLocalRingArea(localRing = []) {
      const ring = Array.isArray(localRing) ? localRing : [];
      if (ring.length < 3) return 0;

      let area = 0;
      for (let index = 0; index < ring.length; index += 1) {
        const current = ring[index];
        const next = ring[(index + 1) % ring.length];
        area +=
          Number(current.x) * Number(next.z) -
          Number(next.x) * Number(current.z);
      }
      return Math.abs(area) / 2;
    },

    isStandaloneBuildingFootprintReasonable(localOuterRing = []) {
      const footprintBounds = this.getLocalRingBounds(localOuterRing);
      if (!footprintBounds) return false;

      const width = Math.abs(
        Number(footprintBounds.maxX) - Number(footprintBounds.minX),
      );
      const depth = Math.abs(
        Number(footprintBounds.maxZ) - Number(footprintBounds.minZ),
      );
      const maximumDimension = Math.max(width, depth);
      const area = this.getLocalRingArea(localOuterRing);

      return (
        maximumDimension <=
          THREE_PARK_BUILDING_MAX_FOOTPRINT_DIMENSION_METERS &&
        area <= THREE_PARK_BUILDING_MAX_FOOTPRINT_AREA_SQUARE_METERS
      );
    },

    createInternalBuildingBoxFootprint(
      localCentroid,
      localOuterRing = [],
      localParts = [],
    ) {
      if (
        !localCentroid ||
        !this.isTerrainPointInsideAnyPart(localCentroid, localParts)
      ) {
        return null;
      }

      const footprintBounds = this.getLocalRingBounds(localOuterRing);
      const width = footprintBounds
        ? Math.abs(Number(footprintBounds.maxX) - Number(footprintBounds.minX))
        : MAPBOX_BUILDING_POINT_FOOTPRINT_METERS;
      const depth = footprintBounds
        ? Math.abs(Number(footprintBounds.maxZ) - Number(footprintBounds.minZ))
        : MAPBOX_BUILDING_POINT_FOOTPRINT_METERS;
      const halfWidth = Math.max(4, Math.min(22, width / 2 || 9));
      const halfDepth = Math.max(4, Math.min(22, depth / 2 || 9));
      let scale = 1;

      for (let attempt = 0; attempt < 7; attempt += 1) {
        const ring = [
          {
            x: Number(localCentroid.x) - halfWidth * scale,
            z: Number(localCentroid.z) - halfDepth * scale,
          },
          {
            x: Number(localCentroid.x) + halfWidth * scale,
            z: Number(localCentroid.z) - halfDepth * scale,
          },
          {
            x: Number(localCentroid.x) + halfWidth * scale,
            z: Number(localCentroid.z) + halfDepth * scale,
          },
          {
            x: Number(localCentroid.x) - halfWidth * scale,
            z: Number(localCentroid.z) + halfDepth * scale,
          },
        ];

        if (
          ring.every((point) =>
            this.isTerrainPointInsideAnyPart(point, localParts),
          )
        ) {
          return ring;
        }

        scale *= 0.68;
      }

      return [
        { x: Number(localCentroid.x) - 3, z: Number(localCentroid.z) - 3 },
        { x: Number(localCentroid.x) + 3, z: Number(localCentroid.z) - 3 },
        { x: Number(localCentroid.x) + 3, z: Number(localCentroid.z) + 3 },
        { x: Number(localCentroid.x) - 3, z: Number(localCentroid.z) + 3 },
      ];
    },

    getNearestTerrainHeight(
      pointMap,
      point,
      minimumElevation,
      verticalExaggeration,
    ) {
      if (!pointMap || !point) return 0;

      let nearest = null;
      let nearestDistance = Infinity;
      pointMap.forEach((record) => {
        const dx = Number(record.x) - Number(point.x);
        const dz = Number(record.z) - Number(point.z);
        const distance = dx * dx + dz * dz;
        if (distance < nearestDistance) {
          nearestDistance = distance;
          nearest = record;
        }
      });

      const elevation =
        nearest && Number.isFinite(Number(nearest.elevation))
          ? Number(nearest.elevation)
          : minimumElevation;

      return (
        (elevation - minimumElevation) *
        Math.max(1, Number(verticalExaggeration) || 1)
      );
    },

    getStandaloneBuildingBaseTerrainHeight(
      pointMap,
      localOuterRing = [],
      parkAnchor,
      minimumElevation,
      verticalExaggeration,
      localParts = [],
    ) {
      const samples = [];
      const pushSample = (point) => {
        if (
          !point ||
          !Number.isFinite(Number(point.x)) ||
          !Number.isFinite(Number(point.z))
        ) {
          return;
        }
        if (
          Array.isArray(localParts) &&
          localParts.length &&
          !this.isTerrainPointInsideAnyPart(point, localParts)
        ) {
          return;
        }
        samples.push(point);
      };

      pushSample(parkAnchor);
      (Array.isArray(localOuterRing) ? localOuterRing : []).forEach(pushSample);
      this.getTerrainRingEdges(localOuterRing).forEach(([a, b]) => {
        pushSample({
          x: (Number(a.x) + Number(b.x)) / 2,
          z: (Number(a.z) + Number(b.z)) / 2,
        });
      });

      let highest = -Infinity;
      samples.forEach((point) => {
        const terrainY = this.getNearestTerrainHeight(
          pointMap,
          point,
          minimumElevation,
          verticalExaggeration,
        );
        if (Number.isFinite(Number(terrainY))) {
          highest = Math.max(highest, Number(terrainY));
        }
      });

      if (Number.isFinite(highest)) return highest;
      return this.getNearestTerrainHeight(
        pointMap,
        parkAnchor,
        minimumElevation,
        verticalExaggeration,
      );
    },

    getSourceFeatureUniqueKey(feature, centroid, height) {
      return [
        feature && (feature.id || feature.properties?.id)
          ? String(feature.id || feature.properties.id)
          : "building",
        Number(centroid && centroid[0]).toFixed(6),
        Number(centroid && centroid[1]).toFixed(6),
        Number(height || 0).toFixed(1),
      ].join("|");
    },

    getBuildingLayerIdsForStandaloneQuery() {
      if (!this.$map) return [];

      // Use only the custom Mapbox Streets building layers that we control.
      // Dynamic Mapbox Standard layers can include buildings from surrounding
      // tiles, which caused white blocks to appear outside the park island.
      return [
        MAPBOX_3D_BUILDINGS_LAYER_ID,
        MAPBOX_3D_BUILDINGS_SAMPLING_LAYER_ID,
      ].filter((layerId) => this.$map.getLayer(layerId));
    },
    getStandaloneBuildingQueryGeometry(origin, localParts = []) {
      if (!origin || !Array.isArray(localParts) || !localParts.length)
        return null;

      const closeRing = (ring = []) => {
        const coordinates = (Array.isArray(ring) ? ring : [])
          .map((point) => this.terrainMetersToLngLat(point, origin))
          .filter(
            (coordinate) =>
              Array.isArray(coordinate) &&
              coordinate.length >= 2 &&
              coordinate.every((value) => Number.isFinite(Number(value))),
          );
        if (coordinates.length < 3) return null;

        const first = coordinates[0];
        const last = coordinates[coordinates.length - 1];
        if (
          Math.abs(Number(first[0]) - Number(last[0])) > 1e-10 ||
          Math.abs(Number(first[1]) - Number(last[1])) > 1e-10
        ) {
          coordinates.push([...first]);
        }
        return coordinates;
      };

      const polygons = localParts
        .map((part) => {
          const outer = closeRing(part.outer);
          if (!outer) return null;
          const holes = (part.holes || []).map(closeRing).filter(Boolean);
          return [outer, ...holes];
        })
        .filter(Boolean);

      if (!polygons.length) return null;
      return polygons.length === 1
        ? { type: "Polygon", coordinates: polygons[0] }
        : { type: "MultiPolygon", coordinates: polygons };
    },

    getStandaloneBuildingLayerFilter(origin, localParts = []) {
      const geometry = this.getStandaloneBuildingQueryGeometry(
        origin,
        localParts,
      );
      return geometry ? ["within", geometry] : null;
    },

    applyStandaloneBuildingSamplingFilter(origin, localParts = []) {
      if (
        !this.$map ||
        !this.$map.getLayer(MAPBOX_3D_BUILDINGS_SAMPLING_LAYER_ID)
      ) {
        return null;
      }

      const filter = this.getStandaloneBuildingLayerFilter(origin, localParts);
      try {
        this.$map.setFilter(
          MAPBOX_3D_BUILDINGS_SAMPLING_LAYER_ID,
          filter || null,
        );
      } catch (error) {
        return null;
      }
      return filter;
    },

    getScreenQueryBoxForBounds(bounds, padding = 96) {
      if (!this.$map || !bounds) return null;

      try {
        const corners = [
          [bounds.minLng, bounds.minLat],
          [bounds.minLng, bounds.maxLat],
          [bounds.maxLng, bounds.minLat],
          [bounds.maxLng, bounds.maxLat],
        ]
          .map((coordinate) => this.$map.project(coordinate))
          .filter(
            (point) =>
              point &&
              Number.isFinite(Number(point.x)) &&
              Number.isFinite(Number(point.y)),
          );

        const canvas = this.$map.getCanvas();
        if (!corners.length || !canvas) return null;

        const xs = corners.map((point) => Number(point.x));
        const ys = corners.map((point) => Number(point.y));
        const minX = Math.max(0, Math.floor(Math.min(...xs) - padding));
        const minY = Math.max(0, Math.floor(Math.min(...ys) - padding));
        const maxX = Math.min(
          canvas.clientWidth,
          Math.ceil(Math.max(...xs) + padding),
        );
        const maxY = Math.min(
          canvas.clientHeight,
          Math.ceil(Math.max(...ys) + padding),
        );

        if (maxX <= minX || maxY <= minY) return null;
        return [
          [minX, minY],
          [maxX, maxY],
        ];
      } catch (error) {
        return null;
      }
    },

    getStandaloneBuildingScreenSampleBoxes(
      bounds,
      origin = null,
      localParts = [],
      pixelRadius = 72,
    ) {
      if (!this.$map || !bounds || !origin || !Array.isArray(localParts)) {
        return [];
      }

      const samples = this.getStandaloneBuildingTilequerySampleCoordinates(
        bounds,
        origin,
        localParts,
      );
      const boxes = [];
      const seen = new Set();
      const canvas = this.$map.getCanvas && this.$map.getCanvas();
      const width = Number(canvas && canvas.clientWidth) || 0;
      const height = Number(canvas && canvas.clientHeight) || 0;
      if (!width || !height) return [];

      samples.forEach((coordinate) => {
        try {
          const point = this.$map.project(coordinate);
          if (
            !point ||
            !Number.isFinite(Number(point.x)) ||
            !Number.isFinite(Number(point.y))
          ) {
            return;
          }

          const minX = Math.max(0, Math.floor(Number(point.x) - pixelRadius));
          const minY = Math.max(0, Math.floor(Number(point.y) - pixelRadius));
          const maxX = Math.min(
            width,
            Math.ceil(Number(point.x) + pixelRadius),
          );
          const maxY = Math.min(
            height,
            Math.ceil(Number(point.y) + pixelRadius),
          );
          if (maxX <= minX || maxY <= minY) return;

          const key = `${minX}|${minY}|${maxX}|${maxY}`;
          if (seen.has(key)) return;
          seen.add(key);
          boxes.push([
            [minX, minY],
            [maxX, maxY],
          ]);
        } catch (error) {
          // Ignore samples that cannot be projected with the current map camera.
        }
      });

      return boxes;
    },

    getStandaloneBuildingSamplingCenters(
      bounds,
      origin = null,
      localParts = [],
    ) {
      if (!bounds) return [];

      const minLng = Number(bounds.minLng);
      const maxLng = Number(bounds.maxLng);
      const minLat = Number(bounds.minLat);
      const maxLat = Number(bounds.maxLat);
      if (![minLng, maxLng, minLat, maxLat].every(Number.isFinite)) return [];

      const centerLng = (minLng + maxLng) / 2;
      const centerLat = (minLat + maxLat) / 2;
      const latitudeScale = 111320;
      const longitudeScale =
        latitudeScale * Math.max(0.15, Math.cos((centerLat * Math.PI) / 180));
      const widthMeters = Math.max(1, (maxLng - minLng) * longitudeScale);
      const heightMeters = Math.max(1, (maxLat - minLat) * latitudeScale);
      const canvas = this.$map && this.$map.getCanvas && this.$map.getCanvas();
      const canvasWidth = Math.max(
        320,
        Number(canvas && canvas.clientWidth) || 900,
      );
      const canvasHeight = Math.max(
        320,
        Number(canvas && canvas.clientHeight) || 700,
      );
      const metersPerPixel =
        (156543.03392 * Math.max(0.15, Math.cos((centerLat * Math.PI) / 180))) /
        Math.pow(2, MAPBOX_3D_BUILDINGS_CAMERA_ZOOM);
      const sampleWidthMeters = Math.max(
        120,
        canvasWidth *
          metersPerPixel *
          THREE_PARK_BUILDING_QUERY_VIEWPORT_COVERAGE_RATIO,
      );
      const sampleHeightMeters = Math.max(
        120,
        canvasHeight *
          metersPerPixel *
          THREE_PARK_BUILDING_QUERY_VIEWPORT_COVERAGE_RATIO,
      );
      const columns = Math.max(
        1,
        Math.min(
          THREE_PARK_BUILDING_QUERY_GRID_MAX_COLUMNS,
          Math.ceil(widthMeters / sampleWidthMeters),
        ),
      );
      const rows = Math.max(
        1,
        Math.min(
          THREE_PARK_BUILDING_QUERY_GRID_MAX_ROWS,
          Math.ceil(heightMeters / sampleHeightMeters),
        ),
      );
      const centers = [];
      const seen = new Set();
      const canFilterByPark =
        origin && Array.isArray(localParts) && localParts.length > 0;
      const pushCenter = (lng, lat, { allowOutsideFallback = false } = {}) => {
        const safeLng = Math.max(minLng, Math.min(maxLng, Number(lng)));
        const safeLat = Math.max(minLat, Math.min(maxLat, Number(lat)));
        if (
          canFilterByPark &&
          !allowOutsideFallback &&
          !this.isLngLatInsideTerrainParts(
            [safeLng, safeLat],
            origin,
            localParts,
          )
        ) {
          return;
        }
        const key = `${safeLng.toFixed(7)}|${safeLat.toFixed(7)}`;
        if (seen.has(key)) return;
        seen.add(key);
        centers.push([safeLng, safeLat]);
      };

      pushCenter(centerLng, centerLat);

      for (let row = 0; row < rows; row += 1) {
        const lat =
          rows === 1
            ? centerLat
            : minLat + ((row + 0.5) / rows) * (maxLat - minLat);
        for (let column = 0; column < columns; column += 1) {
          const lng =
            columns === 1
              ? centerLng
              : minLng + ((column + 0.5) / columns) * (maxLng - minLng);
          pushCenter(lng, lat);
        }
      }

      if (canFilterByPark) {
        this.getStandaloneBuildingTilequerySampleCoordinates(
          bounds,
          origin,
          localParts,
        ).forEach(([lng, lat]) => pushCenter(lng, lat));
      }

      if (!centers.length) {
        pushCenter(centerLng, centerLat, { allowOutsideFallback: true });
      }

      return centers;
    },

    getStandaloneBuildingTilequerySampleCoordinates(
      bounds,
      origin,
      localParts = [],
    ) {
      if (
        !bounds ||
        !origin ||
        !Array.isArray(localParts) ||
        !localParts.length
      ) {
        return [];
      }

      const allPoints = [];
      localParts.forEach((part) => {
        [part.outer, ...(part.holes || [])].forEach((ring) => {
          (Array.isArray(ring) ? ring : []).forEach((point) => {
            if (
              point &&
              Number.isFinite(Number(point.x)) &&
              Number.isFinite(Number(point.z))
            ) {
              allPoints.push(point);
            }
          });
        });
      });
      if (!allPoints.length) return [];

      const xs = allPoints.map((point) => Number(point.x));
      const zs = allPoints.map((point) => Number(point.z));
      const minX = Math.min(...xs);
      const maxX = Math.max(...xs);
      const minZ = Math.min(...zs);
      const maxZ = Math.max(...zs);
      const width = Math.max(1, maxX - minX);
      const depth = Math.max(1, maxZ - minZ);
      const columns = Math.max(
        4,
        Math.min(
          THREE_PARK_BUILDING_QUERY_GRID_MAX_COLUMNS,
          Math.ceil(width / 85),
        ),
      );
      const rows = Math.max(
        4,
        Math.min(
          THREE_PARK_BUILDING_QUERY_GRID_MAX_ROWS,
          Math.ceil(depth / 85),
        ),
      );
      const samples = [];
      const seen = new Set();
      const addLocalPoint = (point) => {
        if (!this.isTerrainPointInsideAnyPart(point, localParts)) return;
        const lngLat = this.terrainMetersToLngLat(point, origin);
        if (
          !Array.isArray(lngLat) ||
          lngLat.length < 2 ||
          !lngLat.every((value) => Number.isFinite(Number(value)))
        ) {
          return;
        }
        if (
          lngLat[0] < bounds.minLng ||
          lngLat[0] > bounds.maxLng ||
          lngLat[1] < bounds.minLat ||
          lngLat[1] > bounds.maxLat
        ) {
          return;
        }
        const key = `${Number(lngLat[0]).toFixed(6)}|${Number(lngLat[1]).toFixed(6)}`;
        if (seen.has(key)) return;
        seen.add(key);
        samples.push(lngLat);
      };

      localParts.forEach((part) => {
        const boundsForPart = this.getLocalRingBounds(part.outer);
        if (!boundsForPart) return;
        addLocalPoint({
          x: (boundsForPart.minX + boundsForPart.maxX) / 2,
          z: (boundsForPart.minZ + boundsForPart.maxZ) / 2,
        });

        this.getTerrainRingEdges(part.outer).forEach(([a, b], index) => {
          if (index % 3 === 0) addLocalPoint(a);
          addLocalPoint({
            x: (Number(a.x) + Number(b.x)) / 2,
            z: (Number(a.z) + Number(b.z)) / 2,
          });
        });
      });

      for (let row = 0; row < rows; row += 1) {
        const z = minZ + ((row + 0.5) / rows) * (maxZ - minZ);
        for (let column = 0; column < columns; column += 1) {
          const x = minX + ((column + 0.5) / columns) * (maxX - minX);
          addLocalPoint({ x, z });
        }
      }

      return samples.slice(0, THREE_PARK_BUILDING_TILEQUERY_MAX_POINTS);
    },

    async fetchMapboxBuildingTilequeryFeatures(coordinate) {
      if (
        typeof fetch !== "function" ||
        !Array.isArray(coordinate) ||
        coordinate.length < 2
      ) {
        return [];
      }

      const lng = Number(coordinate[0]);
      const lat = Number(coordinate[1]);
      if (!Number.isFinite(lng) || !Number.isFinite(lat)) return [];

      const params = new URLSearchParams({
        access_token: MAPBOX_ACCESS_TOKEN,
        layers: MAPBOX_BUILDING_QUERY_SOURCE_LAYERS.join(","),
        radius: String(THREE_PARK_BUILDING_TILEQUERY_RADIUS_METERS),
        limit: String(THREE_PARK_BUILDING_TILEQUERY_LIMIT),
      });
      const url = `https://api.mapbox.com/v4/${MAPBOX_BUILDING_TILEQUERY_TILESET_ID}/tilequery/${lng},${lat}.json?${params.toString()}`;
      const controller =
        typeof AbortController !== "undefined" ? new AbortController() : null;
      const timeoutId = controller
        ? window.setTimeout(
            () => controller.abort(),
            THREE_PARK_BUILDING_TILEQUERY_TIMEOUT_MS,
          )
        : null;

      try {
        const response = await fetch(url, {
          signal: controller ? controller.signal : undefined,
        });
        if (!response || !response.ok) return [];
        const data = await response.json();
        return Array.isArray(data && data.features) ? data.features : [];
      } catch (error) {
        return [];
      } finally {
        if (timeoutId != null) window.clearTimeout(timeoutId);
      }
    },

    async queryMapboxTilequeryBuildingFeaturesForPark(
      bounds,
      origin,
      localParts = [],
    ) {
      const samples = this.getStandaloneBuildingTilequerySampleCoordinates(
        bounds,
        origin,
        localParts,
      );
      if (!samples.length) return [];

      const collected = [];
      for (
        let index = 0;
        index < samples.length;
        index += THREE_PARK_BUILDING_TILEQUERY_BATCH_SIZE
      ) {
        if (!this.$map || !this.shouldShowThreeParkLayer()) break;

        const batch = samples.slice(
          index,
          index + THREE_PARK_BUILDING_TILEQUERY_BATCH_SIZE,
        );
        const batchResults = await Promise.all(
          batch.map((coordinate) =>
            this.fetchMapboxBuildingTilequeryFeatures(coordinate),
          ),
        );
        batchResults.forEach((features) => {
          (Array.isArray(features) ? features : []).forEach((feature) => {
            if (feature && feature.geometry) collected.push(feature);
          });
        });
      }

      return collected;
    },

    queryCurrentMapboxBuildingFeaturesForStandalone3D(
      bounds = null,
      origin = null,
      localParts = [],
    ) {
      if (!this.$map) return [];

      this.ensureStandaloneBuildingSource();
      this.addCustom3DBuildingsLayer();
      const buildingFilter = this.applyStandaloneBuildingSamplingFilter(
        origin,
        localParts,
      );

      const collected = [];
      const addFeatures = (features) => {
        (Array.isArray(features) ? features : []).forEach((feature) => {
          if (feature && feature.geometry) collected.push(feature);
        });
      };

      // Source-feature query from the custom Mapbox Streets building source.
      // It is much faster than camera-scanning the whole park and avoids pulling
      // buildings from unrelated Mapbox Standard 3D layers.
      if (this.$map.getSource(MAPBOX_3D_BUILDINGS_SOURCE_ID)) {
        try {
          const queryOptions = { sourceLayer: "building" };
          if (buildingFilter) queryOptions.filter = buildingFilter;
          addFeatures(
            this.$map.querySourceFeatures(
              MAPBOX_3D_BUILDINGS_SOURCE_ID,
              queryOptions,
            ),
          );
        } catch (error) {
          // Rendered feature query below is used as a fallback.
        }
      }

      try {
        const layers = this.getBuildingLayerIdsForStandaloneQuery();
        const queryBox = this.getScreenQueryBoxForBounds(bounds, 48);
        if (layers.length && queryBox) {
          addFeatures(
            this.$map.queryRenderedFeatures(queryBox, {
              layers,
            }),
          );
        }
      } catch (error) {
        // Some Mapbox styles/layers may not support rendered feature querying.
      }

      return collected;
    },
    async ensureStandaloneBuildingTilesForPark(
      bounds,
      origin = null,
      localParts = [],
    ) {
      if (!this.$map || !bounds) return;

      this.standaloneBuildingFeatureCache = [];
      this.ensureStandaloneBuildingSource();
      this.addCustom3DBuildingsLayer();
      this.applyStandaloneBuildingSamplingFilter(origin, localParts);

      // Keep this fast: do not pan/zoom the hidden Mapbox camera across the
      // whole project and do not call Tilequery. The previous deep scan made
      // "Building 3D Terrain" take several minutes and still captured exterior
      // buildings. One loaded-tile pass is enough; the strict polygon filter in
      // queryStandaloneBuildingFootprints removes anything outside the park.
      try {
        await this.waitForStandaloneBuildingTiles(900);
        await new Promise((resolve) =>
          window.setTimeout(resolve, THREE_PARK_BUILDING_TILE_SETTLE_MS),
        );
        const features = this.queryCurrentMapboxBuildingFeaturesForStandalone3D(
          bounds,
          origin,
          localParts,
        );
        if (features.length) {
          this.standaloneBuildingFeatureCache.push(...features);
        }
      } catch (error) {
        // Building capture is optional. The terrain island should still render
        // quickly even when Mapbox building tiles are unavailable.
      }
    },
    getBuildingSourceFeaturesForStandalone3D(
      bounds = null,
      origin = null,
      localParts = [],
    ) {
      const cached = Array.isArray(this.standaloneBuildingFeatureCache)
        ? this.standaloneBuildingFeatureCache
        : [];
      const current = this.queryCurrentMapboxBuildingFeaturesForStandalone3D(
        bounds,
        origin,
        localParts,
      );
      return cached.concat(current);
    },

    queryStandaloneBuildingFootprints(
      bounds,
      origin,
      localParts,
      pointMap,
      minimumElevation,
      verticalExaggeration,
    ) {
      if (
        !bounds ||
        !origin ||
        !Array.isArray(localParts) ||
        !localParts.length
      ) {
        return [];
      }

      const features = this.getBuildingSourceFeaturesForStandalone3D(
        bounds,
        origin,
        localParts,
      );
      const buildings = [];
      const seen = new Set();

      features.forEach((feature) => {
        const properties = feature.properties || {};
        const polygons = this.getFeatureCoordinateRings(feature);
        if (!polygons.length) return;

        const height = this.getBuildingHeightFromProperties(properties);
        const minHeight = this.getBuildingMinHeightFromProperties(properties);
        const exaggeratedHeight = Math.max(
          MAPBOX_3D_BUILDING_MIN_HEIGHT,
          height * MAPBOX_3D_BUILDING_HEIGHT_EXAGGERATION,
        );

        polygons.forEach((polygon) => {
          const outerCoordinates = this.normalizeTerrainRing(polygon[0]);
          if (outerCoordinates.length < 3) return;

          const centroid = this.getRingCentroid(outerCoordinates);
          if (!centroid) return;

          if (
            centroid[0] < bounds.minLng ||
            centroid[0] > bounds.maxLng ||
            centroid[1] < bounds.minLat ||
            centroid[1] > bounds.maxLat
          ) {
            return;
          }

          let outer = outerCoordinates.map((coordinate) =>
            this.lngLatToTerrainMeters(coordinate, origin),
          );
          if (!this.isStandaloneBuildingFootprintReasonable(outer)) return;

          const localCentroid = this.lngLatToTerrainMeters(centroid, origin);
          const parkAnchor = this.getStandaloneBuildingParkAnchor(
            outer,
            localCentroid,
            localParts,
          );

          // Only keep buildings whose Mapbox footprint intersects the current
          // project polygon. The anchor is the point used to place the building
          // on the island terrain when Mapbox returns a clipped footprint.
          if (
            !parkAnchor ||
            !this.doesStandaloneBuildingBelongToPark(
              outer,
              localCentroid,
              localParts,
            )
          ) {
            return;
          }

          const insideRatio = this.getTerrainRingInsideRatio(outer, localParts);
          const footprintFullyInside = outer.every((point) =>
            this.isTerrainPointInsideAnyPart(point, localParts),
          );

          // Do not snap exterior/boundary buildings onto the floating island.
          // If Mapbox returns a building that is not fully inside the park GIS
          // geometry, skip it so no outside buildings appear in 3D view.
          if (
            !footprintFullyInside ||
            insideRatio < THREE_PARK_BUILDING_MIN_INSIDE_RATIO
          ) {
            return;
          }

          const uniqueKey = this.getSourceFeatureUniqueKey(
            feature,
            centroid,
            exaggeratedHeight,
          );
          if (seen.has(uniqueKey)) return;
          seen.add(uniqueKey);

          const holes = (polygon || [])
            .slice(1)
            .map((ring) => this.normalizeTerrainRing(ring))
            .filter((ring) => ring.length >= 3)
            .map((ring) =>
              ring.map((coordinate) =>
                this.lngLatToTerrainMeters(coordinate, origin),
              ),
            );
          const terrainY = this.getStandaloneBuildingBaseTerrainHeight(
            pointMap,
            outer,
            parkAnchor,
            minimumElevation,
            verticalExaggeration,
            localParts,
          );
          const baseY =
            terrainY + minHeight * MAPBOX_3D_BUILDING_HEIGHT_EXAGGERATION + 0.7;

          buildings.push({
            outer,
            holes,
            baseY,
            topY: baseY + exaggeratedHeight,
            height: exaggeratedHeight,
            properties: { ...properties },
            label:
              properties.name_en ||
              properties.name ||
              properties.Name ||
              properties.type ||
              properties.class ||
              "Building",
          });
        });
      });

      return buildings
        .sort((a, b) => Number(b.height) - Number(a.height))
        .slice(0, 350);
    },

    addStandaloneBuildingMeshes(group, buildingFootprints = []) {
      if (
        !group ||
        !Array.isArray(buildingFootprints) ||
        !buildingFootprints.length
      ) {
        return;
      }

      const positions = [];
      const linePositions = [];

      const pushVertex = (point, y) => {
        positions.push(Number(point.x), Number(y), Number(point.z));
      };

      const pushLine = (a, ay, b, by) => {
        linePositions.push(Number(a.x), Number(ay), Number(a.z));
        linePositions.push(Number(b.x), Number(by), Number(b.z));
      };

      buildingFootprints.forEach((building) => {
        const outer = Array.isArray(building.outer) ? building.outer : [];
        if (outer.length < 3) return;

        const holes = Array.isArray(building.holes) ? building.holes : [];
        const contour = outer.map(
          (point) => new THREE.Vector2(point.x, point.z),
        );
        const holeVectors = holes.map((ring) =>
          ring.map((point) => new THREE.Vector2(point.x, point.z)),
        );
        const flattened = contour.concat(...holeVectors);
        let faces = THREE.ShapeUtils.triangulateShape(contour, holeVectors);

        if (!faces.length) {
          const reversedContour = contour.slice().reverse();
          faces = THREE.ShapeUtils.triangulateShape(
            reversedContour,
            holeVectors,
          );
        }

        faces.forEach((face) => {
          face.forEach((index) => {
            const point = flattened[index];
            if (point) positions.push(point.x, building.topY, point.y);
          });
        });

        const addWallRing = (ring) => {
          for (let index = 0; index < ring.length; index += 1) {
            const current = ring[index];
            const next = ring[(index + 1) % ring.length];

            pushVertex(current, building.baseY);
            pushVertex(next, building.baseY);
            pushVertex(current, building.topY);

            pushVertex(next, building.baseY);
            pushVertex(next, building.topY);
            pushVertex(current, building.topY);

            pushLine(current, building.baseY, next, building.baseY);
            pushLine(current, building.topY, next, building.topY);
            pushLine(current, building.baseY, current, building.topY);
          }
        };

        addWallRing(outer);
        holes.forEach(addWallRing);
      });

      if (positions.length) {
        const geometry = new THREE.BufferGeometry();
        geometry.setAttribute(
          "position",
          new THREE.Float32BufferAttribute(positions, 3),
        );
        geometry.computeVertexNormals();
        const material = new THREE.MeshStandardMaterial({
          color: 0x26383f,
          emissive: 0x07171c,
          emissiveIntensity: 0.2,
          roughness: 0.86,
          metalness: 0.02,
          transparent: false,
          opacity: 1,
          side: THREE.DoubleSide,
        });
        const mesh = new THREE.Mesh(geometry, material);
        mesh.renderOrder = 5;
        group.add(mesh);
      }

      if (linePositions.length) {
        const lineGeometry = new THREE.BufferGeometry();
        lineGeometry.setAttribute(
          "position",
          new THREE.Float32BufferAttribute(linePositions, 3),
        );
        const lineMaterial = new THREE.LineBasicMaterial({
          color: 0x63dce4,
          transparent: true,
          opacity: 0.18,
        });
        const outlines = new THREE.LineSegments(lineGeometry, lineMaterial);
        outlines.renderOrder = 6;
        group.add(outlines);
      }
    },

    getTerrainPointHeight(
      pointMap,
      point,
      minimumElevation,
      verticalExaggeration = 1,
    ) {
      const key = this.getTerrainPointKey(point);
      const record = pointMap.get(key);
      const elevation =
        record && Number.isFinite(Number(record.elevation))
          ? Number(record.elevation)
          : minimumElevation;

      return (
        (elevation - minimumElevation) *
        Math.max(1, Number(verticalExaggeration) || 1)
      );
    },

    createStandaloneTerrainGroup(modelData) {
      const {
        localParts,
        baseTriangles,
        triangleVertexKeys,
        pointMap,
        minimumElevation,
        maximumElevation,
        maxDimension,
        islandWallDepth,
        gridSpacing,
        verticalExaggeration,
        localBounds,
        satelliteTexture,
        buildingFootprints,
      } = modelData;
      const group = new THREE.Group();
      const shouldRenderTopSurface = modelData.showTopSurface !== false;
      const terrainTexture =
        satelliteTexture || this.createStandaloneFallbackTexture();
      const shouldUseSatelliteGround = Boolean(
        satelliteTexture && this.threeLayerVisibility.satelliteGround,
      );
      const textureWidth = Math.max(
        0.0001,
        Number(localBounds.maxX) - Number(localBounds.minX),
      );
      const textureDepth = Math.max(
        0.0001,
        Number(localBounds.maxZ) - Number(localBounds.minZ),
      );

      // Allocate compact typed arrays directly. Building giant normal JS arrays
      // first and then copying them into Float32BufferAttribute temporarily
      // doubles memory usage during large LandXML terrain creation.
      const surfaceVertexCount = triangleVertexKeys.length;
      const surfacePositions = new Float32Array(surfaceVertexCount * 3);
      const surfaceUvs = new Float32Array(surfaceVertexCount * 2);
      for (let vertexIndex = 0; vertexIndex < surfaceVertexCount; vertexIndex += 1) {
        const key = triangleVertexKeys[vertexIndex];
        const point = pointMap.get(key);
        const height =
          (Number(point.elevation) - minimumElevation) * verticalExaggeration;
        const positionOffset = vertexIndex * 3;
        surfacePositions[positionOffset] = Number(point.x);
        surfacePositions[positionOffset + 1] = height;
        surfacePositions[positionOffset + 2] = Number(point.z);

        const u = (Number(point.x) - Number(localBounds.minX)) / textureWidth;
        // Local -Z points north. Texture V=1 is north/top and V=0 is
        // south/bottom, so the satellite image remains geographically aligned.
        const v = (Number(localBounds.maxZ) - Number(point.z)) / textureDepth;
        const uvOffset = vertexIndex * 2;
        surfaceUvs[uvOffset] = Math.max(0, Math.min(1, u));
        surfaceUvs[uvOffset + 1] = Math.max(0, Math.min(1, v));
      }

      const surfaceGeometry = new THREE.BufferGeometry();
      surfaceGeometry.setAttribute(
        "position",
        new THREE.BufferAttribute(surfacePositions, 3),
      );
      surfaceGeometry.setAttribute(
        "uv",
        new THREE.BufferAttribute(surfaceUvs, 2),
      );
      surfaceGeometry.computeVertexNormals();

      // Render the selected terrain elevation with the existing holographic
      // rectangle grid. Satellite Ground changes only the colour texture; it
      // never changes Terrain (before)/(after) geometry or elevation.
      const surfaceMaterial = new THREE.ShaderMaterial({
        uniforms: {
          uSatelliteMap: { value: terrainTexture },
          uUseSatellite: { value: shouldUseSatelliteGround ? 1 : 0 },
          uGroundTextureRepeat: { value: new THREE.Vector2(1, 1) },
          uBaseColor: {
            value: new THREE.Color(THREE_PARK_SURFACE_COLOR),
          },
          uGridColor: {
            value: new THREE.Color(THREE_PARK_WIREFRAME_COLOR),
          },
          uGridSpacing: {
            value: Math.max(1, Number(gridSpacing) || 25),
          },
          uGridIntensity: {
            value: shouldUseSatelliteGround ? 0.24 : 0.18,
          },
          uSatelliteBrightness: {
            value: shouldUseSatelliteGround ? 1.0 : 0.0,
          },
        },
        vertexShader: `
          varying vec2 vUv;
          varying vec3 vWorldPosition;
          varying vec3 vViewNormal;
          varying vec3 vViewPosition;

          void main() {
            vec4 worldPosition = modelMatrix * vec4(position, 1.0);
            vec4 viewPosition = modelViewMatrix * vec4(position, 1.0);
            vUv = uv;
            vWorldPosition = worldPosition.xyz;
            vViewNormal = normalize(normalMatrix * normal);
            vViewPosition = -viewPosition.xyz;
            gl_Position = projectionMatrix * viewPosition;
          }
        `,
        fragmentShader: `
          uniform sampler2D uSatelliteMap;
          uniform float uUseSatellite;
          uniform vec2 uGroundTextureRepeat;
          uniform vec3 uBaseColor;
          uniform vec3 uGridColor;
          uniform float uGridSpacing;
          uniform float uGridIntensity;
          uniform float uSatelliteBrightness;

          varying vec2 vUv;
          varying vec3 vWorldPosition;
          varying vec3 vViewNormal;
          varying vec3 vViewPosition;

          void main() {
            vec3 satelliteColor = texture2D(
              uSatelliteMap,
              vUv * uGroundTextureRepeat
            ).rgb;
            float luminance = dot(
              satelliteColor,
              vec3(0.2126, 0.7152, 0.0722)
            );
            satelliteColor = mix(
              vec3(luminance),
              satelliteColor,
              0.92
            ) * uSatelliteBrightness;

            vec3 baseColor = mix(
              uBaseColor,
              satelliteColor,
              clamp(uUseSatellite, 0.0, 1.0)
            );

            vec2 scaled = vWorldPosition.xz / uGridSpacing;
            vec2 derivativeWidth = max(fwidth(scaled), vec2(0.0001));
            vec2 gridDistance =
              abs(fract(scaled - 0.5) - 0.5) /
              (derivativeWidth * 0.38);
            float gridLine =
              1.0 - clamp(min(gridDistance.x, gridDistance.y), 0.0, 1.0);
            gridLine = smoothstep(0.12, 0.9, gridLine);

            vec3 normalDirection = normalize(vViewNormal);
            vec3 viewDirection = normalize(vViewPosition);
            float facing = max(
              0.0,
              dot(normalDirection, normalize(vec3(0.32, 0.86, 0.38)))
            );
            float fresnel = pow(
              1.0 - abs(dot(normalDirection, viewDirection)),
              2.2
            );

            vec3 shadedBase = baseColor * (0.78 + facing * 0.22);
            vec3 finalColor = mix(
              shadedBase,
              uGridColor,
              gridLine * uGridIntensity
            );
            finalColor += uGridColor * fresnel * 0.04;
            gl_FragColor = vec4(finalColor, 1.0);
          }
        `,
        side: THREE.DoubleSide,
        extensions: {
          derivatives: true,
        },
      });
      const surfaceMesh = new THREE.Mesh(surfaceGeometry, surfaceMaterial);
      surfaceMesh.name = "terrain-top-surface";
      surfaceMesh.userData.baseGridSpacing = Math.max(
        1,
        Number(gridSpacing) || 25,
      );
      surfaceMesh.renderOrder = 1;
      surfaceMesh.visible = shouldRenderTopSurface;
      group.add(surfaceMesh);

      // WireframeGeometry duplicates every TIN edge. That is extremely expensive
      // for King Abdulaziz Park's survey-dense LandXML and was a major source of
      // startup lag / memory pressure. The shader already draws the visible
      // rectangular terrain grid, so omit topology wireframe on large surfaces.
      const surfaceTriangleCount = Math.floor(surfaceVertexCount / 3);
      const renderTinTopology = surfaceTriangleCount <= 120000;
      if (renderTinTopology) {
        const terrainTopology = new THREE.LineSegments(
          new THREE.WireframeGeometry(surfaceGeometry),
          new THREE.LineBasicMaterial({
            color: THREE_PARK_WIREFRAME_COLOR,
            transparent: true,
            opacity: 0.008,
          }),
        );
        terrainTopology.name = "terrain-top-grid";
        terrainTopology.renderOrder = 2;
        terrainTopology.visible = shouldRenderTopSurface;
        group.add(terrainTopology);
      }

      // Use a shared lower plane so the perimeter becomes a floating-island
      // cutaway. Every wall's upper edge is sampled densely from the DEM, so
      // the outline follows the visible landscape elevation instead of drawing
      // one straight line between only the original GIS corner coordinates.
      const baseY = -Math.max(1, Number(islandWallDepth) || 12);
      const wallPositions = [];
      const wallGridPositions = [];
      const bottomOutlinePositions = [];
      const horizontalWallFractions = Array.from(
        { length: Math.max(1, THREE_PARK_WALL_GRID_HORIZONTAL_DIVISIONS - 1) },
        (_, index) =>
          (index + 1) / Math.max(2, THREE_PARK_WALL_GRID_HORIZONTAL_DIVISIONS),
      );

      localParts.forEach((part) => {
        [part.outer, ...(part.holes || [])].forEach((ring) => {
          const topHeights = ring.map((point) =>
            this.getTerrainPointHeight(
              pointMap,
              point,
              minimumElevation,
              verticalExaggeration,
            ),
          );
          const verticalLineStep = Math.max(
            1,
            THREE_PARK_WALL_GRID_VERTICAL_SAMPLE_STEP,
          );

          for (let index = 0; index < ring.length; index += 1) {
            const nextIndex = (index + 1) % ring.length;
            const current = ring[index];
            const next = ring[nextIndex];
            const currentY = topHeights[index];
            const nextY = topHeights[nextIndex];

            wallPositions.push(
              current.x,
              currentY,
              current.z,
              next.x,
              nextY,
              next.z,
              current.x,
              baseY,
              current.z,

              next.x,
              nextY,
              next.z,
              next.x,
              baseY,
              next.z,
              current.x,
              baseY,
              current.z,
            );

            bottomOutlinePositions.push(
              current.x,
              baseY,
              current.z,
              next.x,
              baseY,
              next.z,
            );

            if (index % verticalLineStep === 0) {
              wallGridPositions.push(
                current.x,
                currentY,
                current.z,
                current.x,
                baseY,
                current.z,
              );
            }

            horizontalWallFractions.forEach((fraction) => {
              const currentGridY = currentY + (baseY - currentY) * fraction;
              const nextGridY = nextY + (baseY - nextY) * fraction;
              wallGridPositions.push(
                current.x,
                currentGridY,
                current.z,
                next.x,
                nextGridY,
                next.z,
              );
            });
          }
        });
      });

      const wallGeometry = new THREE.BufferGeometry();
      wallGeometry.setAttribute(
        "position",
        new THREE.Float32BufferAttribute(wallPositions, 3),
      );
      wallGeometry.computeVertexNormals();
      const wallMaterial = new THREE.MeshStandardMaterial({
        color: THREE_PARK_SIDE_COLOR,
        roughness: 0.98,
        metalness: 0,
        transparent: true,
        opacity: 0.9,
        side: THREE.DoubleSide,
      });
      const wallMesh = new THREE.Mesh(wallGeometry, wallMaterial);
      wallMesh.name = "terrain-island-walls";
      wallMesh.renderOrder = 0;
      group.add(wallMesh);

      const wallGridGeometry = new THREE.BufferGeometry();
      wallGridGeometry.setAttribute(
        "position",
        new THREE.Float32BufferAttribute(wallGridPositions, 3),
      );
      const wallGrid = new THREE.LineSegments(
        wallGridGeometry,
        new THREE.LineBasicMaterial({
          color: THREE_PARK_WIREFRAME_COLOR,
          transparent: true,
          opacity: 0.22,
        }),
      );
      wallGrid.name = "terrain-island-wall-grid";
      wallGrid.renderOrder = 2;
      group.add(wallGrid);

      const bottomOutlineGeometry = new THREE.BufferGeometry();
      bottomOutlineGeometry.setAttribute(
        "position",
        new THREE.Float32BufferAttribute(bottomOutlinePositions, 3),
      );
      const bottomOutline = new THREE.LineSegments(
        bottomOutlineGeometry,
        new THREE.LineBasicMaterial({
          color: THREE_PARK_WIREFRAME_COLOR,
          transparent: true,
          opacity: 0.52,
        }),
      );
      bottomOutline.renderOrder = 3;
      group.add(bottomOutline);

      // For LandXML, avoid duplicating the complete TIN just to draw the flat
      // underside. Build the underside from each zone boundary instead. Older
      // Mapbox-derived terrain still supplies baseTriangles and keeps its path.
      if (Array.isArray(baseTriangles) && baseTriangles.length) {
        const basePositions = [];
        baseTriangles.forEach((triangle) => {
          triangle.forEach((point) => {
            basePositions.push(point.x, baseY, point.z);
          });
        });
        const baseGeometry = new THREE.BufferGeometry();
        baseGeometry.setAttribute(
          "position",
          new THREE.Float32BufferAttribute(basePositions, 3),
        );
        baseGeometry.computeVertexNormals();
        const baseMesh = new THREE.Mesh(
          baseGeometry,
          new THREE.MeshBasicMaterial({
            color: THREE_PARK_BASE_COLOR,
            transparent: true,
            opacity: 0.96,
            side: THREE.DoubleSide,
          }),
        );
        baseMesh.name = "terrain-island-base";
        group.add(baseMesh);
      } else {
        localParts.forEach((part) => {
          if (!part || !Array.isArray(part.outer) || part.outer.length < 3) return;
          const shape = new THREE.Shape();
          part.outer.forEach((point, index) => {
            if (index === 0) shape.moveTo(point.x, point.z);
            else shape.lineTo(point.x, point.z);
          });
          shape.closePath();
          (part.holes || []).forEach((holeRing) => {
            if (!Array.isArray(holeRing) || holeRing.length < 3) return;
            const hole = new THREE.Path();
            holeRing.forEach((point, index) => {
              if (index === 0) hole.moveTo(point.x, point.z);
              else hole.lineTo(point.x, point.z);
            });
            hole.closePath();
            shape.holes.push(hole);
          });
          const baseGeometry = new THREE.ShapeGeometry(shape);
          const position = baseGeometry.getAttribute("position");
          for (let index = 0; index < position.count; index += 1) {
            const x = position.getX(index);
            const z = position.getY(index);
            position.setXYZ(index, x, baseY, z);
          }
          position.needsUpdate = true;
          baseGeometry.computeVertexNormals();
          const baseMesh = new THREE.Mesh(
            baseGeometry,
            new THREE.MeshBasicMaterial({
              color: THREE_PARK_BASE_COLOR,
              transparent: true,
              opacity: 0.96,
              side: THREE.DoubleSide,
            }),
          );
          baseMesh.name = "terrain-island-base";
          group.add(baseMesh);
        });
      }

      this.addStandaloneBuildingMeshes(group, buildingFootprints || []);

      localParts.forEach((part) => {
        [part.outer, ...(part.holes || [])].forEach((ring, ringIndex) => {
          const linePoints = ring.map((point) => {
            const y = this.getTerrainPointHeight(
              pointMap,
              point,
              minimumElevation,
              verticalExaggeration,
            );
            return new THREE.Vector3(point.x, y + 0.1, point.z);
          });
          if (linePoints.length) linePoints.push(linePoints[0].clone());

          const lineGeometry = new THREE.BufferGeometry().setFromPoints(
            linePoints,
          );
          const lineMaterial = new THREE.LineBasicMaterial({
            color:
              ringIndex === 0
                ? THREE_PARK_BOUNDARY_COLOR
                : THREE_PARK_WIREFRAME_COLOR,
            transparent: true,
            opacity: ringIndex === 0 ? 0.78 : 0.42,
          });
          const line = new THREE.Line(lineGeometry, lineMaterial);
          line.renderOrder = 4;
          group.add(line);
        });
      });

      group.userData.maxDimension = maxDimension;
      group.userData.verticalExaggeration = verticalExaggeration;
      group.userData.rawElevationRange =
        Number(maximumElevation) - Number(minimumElevation);
      group.userData.islandWallDepth = islandWallDepth;
      group.userData.hasSatelliteTexture = false;
      group.userData.hasLandscapeData = modelData.hasLandscapeData !== false;
      group.userData.showTopSurface = shouldRenderTopSurface;
      return group;
    },

    syncThreeParkHostBounds() {
      const host = this.$refs.threeParkHost;
      const mapElement = this.$refs.map;
      if (!host || !mapElement) return;

      const rect = mapElement.getBoundingClientRect();
      host.style.left = `${rect.left}px`;
      host.style.top = `${rect.top}px`;
      host.style.width = `${rect.width}px`;
      host.style.height = `${rect.height}px`;
    },

    resizeStandaloneThreeRenderer() {
      if (
        !this.threeRenderer ||
        !this.threeCamera ||
        !this.$refs.threeParkHost
      ) {
        return;
      }

      const host = this.$refs.threeParkHost;
      const width = Math.max(1, host.clientWidth);
      const height = Math.max(1, host.clientHeight);
      this.threeRenderer.setSize(width, height, false);
      this.threeCamera.aspect = width / height;
      this.threeCamera.updateProjectionMatrix();
    },

    normaliseMapBearing(value) {
      const bearing = Number(value) || 0;
      return ((((bearing + 180) % 360) + 360) % 360) - 180;
    },

    captureMapboxViewState() {
      if (!this.$map) return null;

      try {
        const center = this.$map.getCenter();
        return {
          center: [Number(center.lng), Number(center.lat)],
          zoom: Number(this.$map.getZoom()),
          pitch: Number(this.$map.getPitch()),
          bearing: this.normaliseMapBearing(this.$map.getBearing()),
        };
      } catch (error) {
        return null;
      }
    },

    captureStandaloneThreeViewState() {
      if (!this.threeCamera || !this.threeControls) return null;

      const offset = this.threeCamera.position
        .clone()
        .sub(this.threeControls.target);
      const horizontalDistance = Math.hypot(offset.x, offset.z);
      const pitch = THREE.MathUtils.radToDeg(
        Math.atan2(horizontalDistance, Math.max(0.0001, offset.y)),
      );
      const bearing = THREE.MathUtils.radToDeg(Math.atan2(-offset.x, offset.z));
      const base = this.preservedMapViewState || {};

      return {
        ...base,
        pitch: Math.max(0, Math.min(85, pitch)),
        bearing: this.normaliseMapBearing(bearing),
      };
    },

    updateStandaloneThreeCompass() {
      const rose = this.$refs.threeCompassRose;
      if (!rose || !this.threeCamera || !this.threeControls) return;

      const state = this.captureStandaloneThreeViewState();
      const bearing = this.normaliseMapBearing(state ? state.bearing : 0);

      // Map bearing rotates the scene clockwise. Rotate the compass rose in the
      // opposite direction so its N arrow always points toward geographic north
      // on screen, regardless of how the user orbits the floating island.
      rose.style.transform = `rotate(${-bearing}deg)`;
    },

    preserveCurrentViewState() {
      const state = this.isStandaloneThreeMode
        ? this.captureStandaloneThreeViewState()
        : this.captureMapboxViewState();

      if (state) this.preservedMapViewState = state;
      return state;
    },

    getThreeCameraDirectionFromMapView(state, fallbackDirection) {
      if (!state) return fallbackDirection.clone().normalize();

      const pitch = THREE.MathUtils.degToRad(
        Math.max(1, Math.min(85, Number(state.pitch) || 0)),
      );
      const bearing = THREE.MathUtils.degToRad(
        this.normaliseMapBearing(state.bearing),
      );
      const horizontal = Math.sin(pitch);

      return new THREE.Vector3(
        -Math.sin(bearing) * horizontal,
        Math.cos(pitch),
        Math.cos(bearing) * horizontal,
      ).normalize();
    },

    applyMapboxViewState(state = this.preservedMapViewState, duration = 0) {
      if (!this.$map || !state) return;

      const options = {
        pitch: Math.max(0, Math.min(85, Number(state.pitch) || 0)),
        bearing: this.normaliseMapBearing(state.bearing),
        duration,
      };

      if (
        Array.isArray(state.center) &&
        state.center.length >= 2 &&
        state.center.every((value) => Number.isFinite(Number(value)))
      ) {
        options.center = [Number(state.center[0]), Number(state.center[1])];
      }
      if (Number.isFinite(Number(state.zoom))) {
        options.zoom = Number(state.zoom);
      }

      this.$map.easeTo(options);
    },

    getMapboxCameraStateForFeatures(features, maxZoom = 13) {
      if (
        !this.$map ||
        !this.mapboxgl ||
        !Array.isArray(features) ||
        !features.length ||
        typeof this.$map.cameraForBounds !== "function"
      ) {
        return null;
      }

      const bounds = new this.mapboxgl.LngLatBounds();
      let hasBounds = false;
      features.forEach((feature) => {
        if (!feature || !feature.geometry) return;
        this.eachCoordinate(feature.geometry.coordinates, (coord) => {
          if (
            Array.isArray(coord) &&
            coord.length >= 2 &&
            Number.isFinite(Number(coord[0])) &&
            Number.isFinite(Number(coord[1]))
          ) {
            bounds.extend([Number(coord[0]), Number(coord[1])]);
            hasBounds = true;
          }
        });
      });
      if (!hasBounds) return null;

      try {
        const camera = this.$map.cameraForBounds(bounds, {
          padding: 40,
          maxZoom,
          pitch: 0,
          bearing: 0,
        });
        if (!camera) return null;

        const center = camera.center || bounds.getCenter();
        return {
          center: [Number(center.lng), Number(center.lat)],
          zoom: Number.isFinite(Number(camera.zoom))
            ? Number(camera.zoom)
            : Number(maxZoom),
          pitch: 0,
          bearing: 0,
        };
      } catch (error) {
        return null;
      }
    },

    getDefaultResetCameraStateForCurrentMapContext() {
      if (this.activeViewMode === "project" && this.currentProjectId) {
        return this.getMapboxCameraStateForFeatures(
          this.getProjectFeaturesForId(this.currentProjectId),
          14,
        );
      }

      if (
        this.hasActiveSearchFilter() &&
        Array.isArray(this.activeSearchProjectIds) &&
        this.activeSearchProjectIds.length
      ) {
        const ids = this.activeSearchProjectIds
          .map((id) => this.normalizeProjectId(id))
          .filter(Boolean);
        const features = this.allProjectFeatures.filter((feature) =>
          ids.includes(
            this.normalizeProjectId(
              feature.properties.parent_id ?? feature.properties.id,
            ),
          ),
        );
        return this.getMapboxCameraStateForFeatures(features, 13);
      }

      if (this.activeViewMode === "near" && this.nearProjectId) {
        return this.getMapboxCameraStateForFeatures(
          this.getProjectFeaturesForId(this.nearProjectId),
          13,
        );
      }

      return {
        center: this.getDefaultCenter(),
        zoom: Number(this.zoom || 10),
        pitch: 0,
        bearing: 0,
      };
    },

    clearMapCameraRestoreTimers() {
      if (!Array.isArray(this.mapCameraRestoreTimers)) {
        this.mapCameraRestoreTimers = [];
        return;
      }

      this.mapCameraRestoreTimers.forEach((timerId) => {
        window.clearTimeout(timerId);
      });
      this.mapCameraRestoreTimers = [];
    },

    clearSatelliteTerrainRestoreTimers() {
      if (!Array.isArray(this.satelliteTerrainRestoreTimers)) {
        this.satelliteTerrainRestoreTimers = [];
        return;
      }

      this.satelliteTerrainRestoreTimers.forEach((timerId) => {
        window.clearTimeout(timerId);
      });
      this.satelliteTerrainRestoreTimers = [];
    },

    hardRefreshSatelliteTerrain3D() {
      if (
        !this.$map ||
        this.activeBaseMapMode !== "satellite" ||
        this.isStandaloneThreeMode ||
        !this.$map.isStyleLoaded ||
        !this.$map.isStyleLoaded()
      ) {
        return;
      }

      try {
        this.showMapboxRenderer();
        if (typeof this.$map.resize === "function") this.$map.resize();

        if (!this.$map.getSource(MAPBOX_TERRAIN_SOURCE_ID)) {
          this.$map.addSource(MAPBOX_TERRAIN_SOURCE_ID, {
            type: "raster-dem",
            url: "mapbox://mapbox.mapbox-terrain-dem-v1",
            tileSize: 512,
            maxzoom: 14,
          });
        }

        // Force Mapbox to rebuild the terrain mesh. This fixes the state where
        // returning from the standalone 3D reset could leave Satellite imagery
        // draped on a visually flat plane even though buildings were still 3D.
        try {
          this.$map.setTerrain(null);
        } catch (error) {
          // Some style states reject clearing terrain briefly while loading.
        }

        this.$map.setTerrain({
          source: MAPBOX_TERRAIN_SOURCE_ID,
          exaggeration: MAPBOX_3D_TERRAIN_EXAGGERATION,
        });
        this.setMapboxStandard3DOptions();
        this.addCustom3DBuildingsLayer();
        if (typeof this.$map.triggerRepaint === "function") {
          this.$map.triggerRepaint();
        }
      } catch (error) {
        console.warn("Satellite 3D terrain could not be refreshed:", error);
      }
    },

    scheduleSatelliteTerrainRestore({ hardRefresh = false } = {}) {
      this.clearSatelliteTerrainRestoreTimers();
      if (hardRefresh) this.pendingSatelliteTerrainHardRefresh = true;

      [0, 120, 360, 760, 1400, 2300].forEach((delay, index, delays) => {
        const timerId = window.setTimeout(() => {
          if (
            this.activeBaseMapMode === "satellite" &&
            !this.isStandaloneThreeMode &&
            this.$map &&
            this.$map.isStyleLoaded &&
            this.$map.isStyleLoaded()
          ) {
            if (this.pendingSatelliteTerrainHardRefresh || hardRefresh) {
              this.hardRefreshSatelliteTerrain3D();
            } else {
              this.applyThreeDimensionalMapSettings();
            }
          }

          if (index === delays.length - 1) {
            this.pendingSatelliteTerrainHardRefresh = false;
          }
        }, delay);
        this.satelliteTerrainRestoreTimers.push(timerId);
      });
    },

    restoreMapCameraStateImmediately(state = this.pendingBaseMapCameraState) {
      if (!this.$map || !state) return;

      const options = {
        pitch: Math.max(0, Math.min(85, Number(state.pitch) || 0)),
        bearing: this.normaliseMapBearing(state.bearing),
      };

      if (
        Array.isArray(state.center) &&
        state.center.length >= 2 &&
        state.center.every((value) => Number.isFinite(Number(value)))
      ) {
        options.center = [Number(state.center[0]), Number(state.center[1])];
      }
      if (Number.isFinite(Number(state.zoom))) {
        options.zoom = Number(state.zoom);
      }

      try {
        this.$map.stop();
        this.$map.jumpTo(options);
      } catch (error) {
        // The style may still be replacing its internal camera transform. A
        // later scheduled restore will apply the same state again.
      }
    },

    lockCameraAcrossBaseMapStyleChange(state) {
      if (!state) return;

      this.pendingBaseMapCameraState = {
        ...state,
        center: Array.isArray(state.center) ? [...state.center] : state.center,
      };
      this.preservedMapViewState = this.pendingBaseMapCameraState;
      this.isBaseMapStyleSwitching = true;
      this.clearMapCameraRestoreTimers();

      const restore = () => {
        this.restoreMapCameraStateImmediately(this.pendingBaseMapCameraState);
      };

      restore();
      window.requestAnimationFrame(() => {
        restore();
        window.requestAnimationFrame(restore);
      });

      [60, 180, 420, 850].forEach((delay, index, delays) => {
        const timerId = window.setTimeout(() => {
          restore();
          if (index === delays.length - 1) {
            this.isBaseMapStyleSwitching = false;
            this.pendingBaseMapCameraState = null;
            this.clearMapCameraRestoreTimers();
          }
        }, delay);
        this.mapCameraRestoreTimers.push(timerId);
      });
    },

    fitStandaloneThreeCamera() {
      if (!this.threeParkGroup || !this.threeCamera || !this.threeControls) {
        return;
      }

      const box = new THREE.Box3().setFromObject(this.threeParkGroup);
      const sphere = new THREE.Sphere();
      box.getBoundingSphere(sphere);
      const radius = Math.max(18, sphere.radius);
      const verticalFov = (Number(this.threeCamera.fov) * Math.PI) / 180;
      const distance =
        (radius / Math.sin(Math.max(0.2, verticalFov / 2))) * 0.9;
      const defaultDirection = new THREE.Vector3(1, 0.9, 1).normalize();
      const direction = this.getThreeCameraDirectionFromMapView(
        this.preservedMapViewState,
        defaultDirection,
      );
      const resetMapViewState =
        this.getDefaultResetCameraStateForCurrentMapContext() || {
          pitch: 0,
          bearing: 0,
        };
      const resetDirection = this.getThreeCameraDirectionFromMapView(
        resetMapViewState,
        defaultDirection,
      );
      const target = sphere.center.clone();
      target.y += radius * 0.03;

      this.threeCamera.near = Math.max(0.02, Math.min(0.2, radius / 4000));
      this.threeCamera.far = Math.max(3000, distance + radius * 12);
      this.threeCamera.position.copy(
        target.clone().add(direction.multiplyScalar(distance)),
      );
      this.threeCamera.updateProjectionMatrix();

      this.threeControls.target.copy(target);
      // The former radius * 0.28 limit stopped users far above the park. This
      // much smaller limit allows wheel-zooming down to an individual tree.
      this.threeControls.minDistance = Math.max(0.9, radius * 0.006);
      this.threeControls.maxDistance = radius * 6;
      this.threeControls.update();

      this.threeDefaultCameraState = {
        position: target.clone().add(resetDirection.multiplyScalar(distance)),
        target: target.clone(),
        mapViewState: resetMapViewState,
      };
    },

    resetThreeParkCamera() {
      if (
        !this.threeDefaultCameraState ||
        !this.threeCamera ||
        !this.threeControls
      ) {
        return;
      }

      this.threeCamera.position.copy(this.threeDefaultCameraState.position);
      this.threeControls.target.copy(this.threeDefaultCameraState.target);
      this.threeControls.update();

      if (this.threeDefaultCameraState.mapViewState) {
        this.preservedMapViewState = {
          ...this.threeDefaultCameraState.mapViewState,
        };
        this.lastStandaloneThreeResetMapViewState = {
          ...this.threeDefaultCameraState.mapViewState,
        };
      }

      // When the user returns to Satellite after resetting the standalone 3D
      // camera, refresh the Mapbox terrain mesh at the configured exaggeration.
      this.pendingSatelliteTerrainHardRefresh = true;
    },

    zoomStandaloneThree(factor) {
      if (!this.threeCamera || !this.threeControls) return;

      const offset = this.threeCamera.position
        .clone()
        .sub(this.threeControls.target)
        .multiplyScalar(factor);
      const distance = offset.length();
      const minimum = Number(this.threeControls.minDistance || 1);
      const maximum = Number(this.threeControls.maxDistance || Infinity);
      const clampedDistance = Math.min(maximum, Math.max(minimum, distance));
      offset.setLength(clampedDistance);
      this.threeCamera.position.copy(
        this.threeControls.target.clone().add(offset),
      );
      this.threeControls.update();
    },

    initialiseStandaloneThreeScene(modelData) {
      this.destroyStandaloneThreeScene();
      this.syncThreeParkHostBounds();

      const host = this.$refs.threeParkHost;
      if (!host) {
        throw new Error("The 3D terrain container is unavailable.");
      }

      const renderer = new THREE.WebGLRenderer({
        antialias: true,
        alpha: false,
        powerPreference: "high-performance",
      });
      renderer.domElement.className = "three-park-canvas";
      renderer.setClearColor(THREE_PARK_BACKGROUND, 1);
      renderer.setPixelRatio(Math.min(Number(window.devicePixelRatio) || 1, 2));
      if (THREE.sRGBEncoding) {
        renderer.outputEncoding = THREE.sRGBEncoding;
      }
      if (THREE.ACESFilmicToneMapping) {
        renderer.toneMapping = THREE.ACESFilmicToneMapping;
        // The previous exposure pushed pale concrete, building outlines and
        // dense tree wireframes toward white. A lower exposure keeps the dark
        // holographic mood of the concept while preserving readable highlights.
        renderer.toneMappingExposure = 0.8;
      }

      const scene = new THREE.Scene();
      scene.background = new THREE.Color(THREE_PARK_BACKGROUND);

      const camera = new THREE.PerspectiveCamera(42, 1, 0.1, 10000);
      const controls = new OrbitControls(camera, renderer.domElement);
      controls.enableDamping = true;
      controls.dampingFactor = 0.08;
      controls.zoomSpeed = THREE_PARK_MOUSE_WHEEL_ZOOM_SPEED;
      controls.zoomToCursor = true;
      controls.enablePan = true;
      // World-plane panning keeps camera/target elevation fixed while the
      // left mouse button navigates horizontally across the park.
      controls.screenSpacePanning = false;
      controls.minPolarAngle = 0.02;
      controls.maxPolarAngle = Math.PI * 0.49;
      controls.mouseButtons = {
        LEFT: THREE.MOUSE.PAN,
        // Middle/right mouse input belongs exclusively to threeCameraControls.js.
        // Keeping OrbitControls disabled for these buttons prevents the legacy
        // orbit/drag behaviour from fighting the custom fixed-position camera.
        MIDDLE: null,
        RIGHT: null,
      };
      controls.touches = {
        ONE: THREE.TOUCH.PAN,
        TWO: THREE.TOUCH.DOLLY_ROTATE,
      };
      controls.addEventListener("start", () => {
        this.lastStandaloneThreeResetMapViewState = null;
      });

      const hemisphereLight = new THREE.HemisphereLight(
        0x8fd8de,
        0x030b0f,
        0.72,
      );
      scene.add(hemisphereLight);

      const keyLight = new THREE.DirectionalLight(0xdffbff, 0.88);
      keyLight.position.set(1.2, 2.4, 1.1);
      scene.add(keyLight);

      const rimLight = new THREE.DirectionalLight(0x32c7d2, 0.38);
      rimLight.position.set(-1.4, 1.1, -0.8);
      scene.add(rimLight);

      const terrainContainer = new THREE.Group();
      terrainContainer.name = "standalone-terrain-container";

      const beforeModelData = modelData.terrainBeforeData || modelData;
      const afterModelData = modelData.terrainAfterData || modelData;

      const terrainBeforeGroup =
        this.createStandaloneTerrainGroup(beforeModelData);
      terrainBeforeGroup.name = "standalone-terrain-before";

      const terrainAfterGroup =
        this.createStandaloneTerrainGroup(afterModelData);
      terrainAfterGroup.name = "standalone-terrain-after";

      terrainContainer.add(terrainBeforeGroup);
      terrainContainer.add(terrainAfterGroup);
      scene.add(terrainContainer);

      host.appendChild(renderer.domElement);

      this.threeRenderer = renderer;
      this.threeScene = scene;
      this.threeCamera = camera;
      this.threeControls = controls;
      this.threeParkGroup = terrainContainer;
      this.threeParkModelData = modelData;
      // All standalone 3D keyboard/right-mouse/middle-mouse camera input is
      // handled by this module. Map.vue must not install a second keyboard
      // listener for WASD/QE/arrow keys.
      this.threeCameraControls = createThreeCameraControls({
        camera,
        controls,
        domElement: renderer.domElement,
        onCameraChanged: () => {
          this.lastStandaloneThreeResetMapViewState = null;
        },
      });
      // Structure labels are populated from the ArcGIS GeoJSON when the
      // Structures layer loads. The GIS department's labels are authoritative.
      this.threeStructureAnnotations = [];
      this.threeLayerGroups = {
        terrain: terrainContainer,
        terrainBefore: terrainBeforeGroup,
        terrainAfter: terrainAfterGroup,
        paths: null,
        water: null,
        structures: null,
        vegetation: null,
        tse: null,
      };
      this.applyStandaloneTerrainDisplayMode(
        this.threeTerrainMode,
        this.threeLayerVisibility.terrain,
      );

      this.resizeStandaloneThreeRenderer();
      this.fitStandaloneThreeCamera();
      this.$nextTick(() => this.updateStandaloneThreeCompass());

      let previousFrameTime = null;
      const animate = (frameTime) => {
        if (!this.threeRenderer || !this.threeScene || !this.threeCamera) {
          return;
        }

        const currentFrameTime = Number.isFinite(Number(frameTime))
          ? Number(frameTime)
          : typeof performance !== "undefined"
            ? performance.now()
            : Date.now();
        const deltaSeconds =
          previousFrameTime == null
            ? 0
            : Math.max(0, (currentFrameTime - previousFrameTime) / 1000);
        previousFrameTime = currentFrameTime;

        this.threeParkAnimationFrame = window.requestAnimationFrame(animate);
        if (this.threeCameraControls) {
          this.threeCameraControls.update(deltaSeconds);
        }
        if (this.threeControls) this.threeControls.update();
        this.updateStandaloneThreeCompass();
        this.updateStandaloneStructureLabelOverlay();
        this.updateStandaloneVegetationLineLod();
        this.threeRenderer.render(this.threeScene, this.threeCamera);
      };
      animate();

      if (typeof ResizeObserver !== "undefined") {
        this.threeParkResizeObserver = new ResizeObserver(() => {
          this.syncThreeParkHostBounds();
          this.resizeStandaloneThreeRenderer();
        });
        this.threeParkResizeObserver.observe(host);
      }
    },

    destroyStandaloneThreeScene() {
      if (this.threeParkAnimationFrame != null) {
        window.cancelAnimationFrame(this.threeParkAnimationFrame);
        this.threeParkAnimationFrame = null;
      }

      if (this.threeParkResizeObserver) {
        this.threeParkResizeObserver.disconnect();
        this.threeParkResizeObserver = null;
      }

      if (this.threeCameraControls) {
        this.threeCameraControls.dispose();
        this.threeCameraControls = null;
      }

      if (this.threeControls) {
        this.threeControls.dispose();
        this.threeControls = null;
      }

      if (this.threeScene) {
        this.threeScene.traverse((object) => {
          if (object.geometry && object.geometry.dispose) {
            object.geometry.dispose();
          }

          const materials = Array.isArray(object.material)
            ? object.material
            : object.material
              ? [object.material]
              : [];

          materials.forEach((material) => {
            Object.keys(material || {}).forEach((key) => {
              const value = material[key];
              if (value && value.isTexture && value.dispose) {
                value.dispose();
              }
            });

            if (material && material.uniforms) {
              Object.keys(material.uniforms).forEach((uniformKey) => {
                const uniform = material.uniforms[uniformKey];
                const value = uniform && uniform.value;
                if (value && value.isTexture && value.dispose) {
                  value.dispose();
                }
              });
            }

            if (material && material.dispose) material.dispose();
          });
        });
      }

      if (this.threeRenderer) {
        const canvas = this.threeRenderer.domElement;
        this.threeRenderer.dispose();
        if (canvas && canvas.parentNode) {
          canvas.parentNode.removeChild(canvas);
        }
      }

      this.threeVegetationLoadToken += 1;
      this.threeSatelliteGroundLoadToken += 1;
      this.threeCompactedGroundLoadToken += 1;
      if (
        this.threeSatelliteGroundTexture &&
        typeof this.threeSatelliteGroundTexture.dispose === "function"
      ) {
        this.threeSatelliteGroundTexture.dispose();
      }
      if (
        this.threeCompactedGroundTexture &&
        typeof this.threeCompactedGroundTexture.dispose === "function"
      ) {
        this.threeCompactedGroundTexture.dispose();
      }
      this.threeSatelliteGroundTexture = null;
      this.threeCompactedGroundTexture = null;
      this.threeStructureAnnotations = [];
      this.threeVisibleStructureLabels = [];
      this.threeToolState.structureLabels = false;
      this.threeToolState.conceptPaths = false;
      this.threeToolState.treeTypes = false;
      this.threeTreeTypeLoading = false;
      this.threeTreeTypeStats = [];
      this.threeRenderer = null;
      this.threeScene = null;
      this.threeCamera = null;
      this.threeParkGroup = null;
      this.threeParkModelData = null;
      this.threeLayerGroups = {
        terrain: null,
        terrainBefore: null,
        terrainAfter: null,
        paths: null,
        water: null,
        structures: null,
        vegetation: null,
        tse: null,
      };
      this.threeDefaultCameraState = null;
      this.threeProjectGisData = null;
      this.threeProjectGisUrl = "";
      this.threeProjectGisFallbackLayer = "";
      this.threeProjectGisLoadError = "";
    },

    suspendMapboxRendererForStandaloneThree() {
      if (!this.$map) return;

      this.suspendMapStyleRestoration = true;
      this.mapboxRendererSuspended = true;
      this.unbindProjectLayerEvents();

      // Do not call map.stop() here. Stopping the hidden Mapbox renderer while
      // terrain tiles are active can leave Satellite mode visually flat after
      // returning from the standalone Three.js view. Hiding the map element is
      // enough because the Three.js canvas receives all pointer events in 3D.

      // Do not remove Mapbox terrain or replace the style with a blank style.
      // Keeping the Satellite style alive prevents the Satellite terrain from
      // returning flat after the user resets the standalone Three.js 3D view.
      const mapElement = this.$refs.map;
      if (mapElement) {
        mapElement.style.visibility = "hidden";
        mapElement.style.pointerEvents = "none";
      }
    },

    async enterStandaloneThreeMode() {
      if (!this.shouldShowThreeParkLayer()) return;

      this.resetStandaloneThreeLayerFilters();

      const existingThreeState = this.captureStandaloneThreeViewState();
      if (existingThreeState) {
        this.preservedMapViewState = existingThreeState;
      }

      const buildToken = this.threeParkBuildToken + 1;
      this.threeParkBuildToken = buildToken;
      this.threeParkLoading = true;
      this.threeParkError = "";
      this.destroyStandaloneThreeScene();

      await new Promise((resolve) => this.$nextTick(resolve));
      this.syncThreeParkHostBounds();

      try {
        // GIS content is optional for constructing the base terrain shell. If
        // the GIS URL is temporarily unavailable, keep building Terrain
        // (before)/(after) instead of replacing the whole 3D view with an error.
        try {
          await this.prepareStandaloneProjectGis();
        } catch (gisError) {
          this.threeProjectGisLoadError =
            gisError && gisError.message
              ? String(gisError.message)
              : "The configured GIS data could not be loaded.";
          console.warn(
            "Standalone GIS data could not be prepared; continuing with the terrain shell.",
            gisError,
          );
        }
        if (
          buildToken !== this.threeParkBuildToken ||
          !this.shouldShowThreeParkLayer()
        ) {
          return;
        }

        const parts = this.getProjectTerrainPolygonParts();
        if (!parts.length) {
          throw new Error(
            "This project does not contain a polygon geometry that can be rendered in 3D.",
          );
        }

        const bounds = this.getTerrainPartsBounds(parts);
        const origin = this.getTerrainOrigin(bounds);
        if (!bounds || !origin) {
          throw new Error("The project geometry coordinates are invalid.");
        }

        // Mapbox DEM sampling improves Terrain (before), but it is not a
        // requirement for opening the 3D project. If DEM setup is unavailable,
        // buildStandaloneTerrainData() below already falls back to a flat shell.
        try {
          await this.ensureTerrainSamplingReady(bounds);
        } catch (terrainSamplingError) {
          console.warn(
            "Mapbox terrain sampling was unavailable; continuing with the flat terrain fallback.",
            terrainSamplingError,
          );
        }
        if (
          buildToken !== this.threeParkBuildToken ||
          !this.shouldShowThreeParkLayer()
        ) {
          return;
        }

        // Build two genuinely different terrain models:
        // - Terrain (before): Mapbox's existing-ground elevation model.
        // - Terrain (after): the Civil 3D LandXML design/as-built surface.
        // Previously buildStandaloneTerrainData() automatically preferred
        // LandXML, so both checkboxes pointed at the same model.
        const terrainBeforeData = await this.buildStandaloneTerrainData(
          parts,
          origin,
          bounds,
          buildToken,
          false,
        );
        if (
          buildToken !== this.threeParkBuildToken ||
          !this.shouldShowThreeParkLayer()
        ) {
          return;
        }

        const landscapeUrls = this.getStandaloneProjectLandscapeUrls();

        // Keep Terrain (after) selectable even when the project has no
        // landscape value, but do not copy the Terrain (before) floor into it.
        // The open top makes the missing proposed/as-built surface obvious to
        // the viewer instead of incorrectly suggesting that after data exists.
        let terrainAfterData = {
          ...terrainBeforeData,
          landscapeUrl: "",
          hasLandscapeData: false,
          showTopSurface: false,
        };

        if (landscapeUrls.length) {
          try {
            const loadedTerrainAfterData =
              await this.buildStandaloneTerrainDataFromLandXml(
                landscapeUrls,
                origin,
                bounds,
                buildToken,
              );

            terrainAfterData = {
              ...loadedTerrainAfterData,
              hasLandscapeData: true,
              showTopSurface: true,
            };
          } catch (error) {
            console.warn(
              "LandXML Terrain (after) could not be used; showing an open Terrain (after) model because no valid after surface is available.",
              error,
            );
          }
        }

        terrainBeforeData.hasLandscapeData = true;
        terrainBeforeData.showTopSurface = true;

        // Mapbox buildings must use the project-polygon clipping produced by
        // Terrain (before). The LandXML boundary can be larger than the park and
        // must never be used to admit surrounding Mapbox buildings.
        terrainAfterData.buildingFootprints = Array.isArray(
          terrainBeforeData.buildingFootprints,
        )
          ? terrainBeforeData.buildingFootprints
          : [];

        const modelData = terrainAfterData;
        modelData.terrainBeforeData = terrainBeforeData;
        modelData.terrainAfterData = terrainAfterData;

        // Satellite Ground is loaded lazily only when its Layer Filters
        // checkbox is enabled. This keeps the normal 3D build fast and leaves
        // Terrain (before)/(after) elevation data completely unchanged.
        terrainBeforeData.satelliteTexture = null;
        terrainAfterData.satelliteTexture = null;
        modelData.satelliteTexture = null;
        if (
          buildToken !== this.threeParkBuildToken ||
          !this.shouldShowThreeParkLayer()
        ) {
          return;
        }

        this.initialiseStandaloneThreeScene(modelData);
        console.info("3D park terrain diagnostics:", {
          hasTerrainData: modelData.hasTerrainData,
          terrainSource: modelData.terrainSource || "mapbox",
          terrainBeforeSource:
            modelData.terrainBeforeData &&
            modelData.terrainBeforeData.terrainSource
              ? modelData.terrainBeforeData.terrainSource
              : "mapbox",
          terrainAfterSource:
            modelData.terrainAfterData &&
            modelData.terrainAfterData.terrainSource
              ? modelData.terrainAfterData.terrainSource
              : modelData.terrainSource || "mapbox",
          landscapeUrl: modelData.landscapeUrl || "",
          hasLandscapeData: Boolean(
            modelData.terrainAfterData &&
            modelData.terrainAfterData.hasLandscapeData,
          ),
          terrainAfterHasTopSurface: Boolean(
            modelData.terrainAfterData &&
            modelData.terrainAfterData.showTopSurface !== false,
          ),
          minimumElevation: modelData.minimumElevation,
          maximumElevation: modelData.maximumElevation,
          rawElevationRange: modelData.rawElevationRange,
          verticalExaggeration: modelData.verticalExaggeration,
          visibleElevationRange: modelData.visibleElevationRange,
          parkMaximumDimensionMeters: modelData.maxDimension,
          islandWallDepthMeters: modelData.islandWallDepth,
          boundarySampleSpacingMeters: modelData.boundarySampleSpacing,
          gridSpacingMeters: modelData.gridSpacing,
          hasSatelliteTexture: false,
          buildingCount: Array.isArray(modelData.buildingFootprints)
            ? modelData.buildingFootprints.length
            : 0,
        });
        this.suspendMapboxRendererForStandaloneThree();
        this.threeParkLoading = false;

        if (!modelData.hasTerrainData) {
          console.warn(
            "Mapbox elevation tiles were unavailable, so the park cutout was rendered with a flat terrain fallback.",
          );
        }
      } catch (error) {
        if (buildToken !== this.threeParkBuildToken) return;
        console.error("Standalone Three.js park terrain failed:", error);
        this.threeParkLoading = false;
        this.threeParkError = this.isArabicUI()
          ? "تعذر إنشاء نموذج التضاريس ثلاثي الأبعاد لهذه الحديقة."
          : "The 3D terrain model could not be created for this park.";
      }
    },

    exitStandaloneThreeMode() {
      this.threeParkBuildToken += 1;
      this.threeVegetationLoadToken += 1;
      this.threeParkLoading = false;
      this.threeParkError = "";
      this.threeLayerLoadingKey = "";
      this.threeLayerError = "";
      this.standaloneBuildingFeatureCache = [];
      this.destroyStandaloneThreeScene();
      this.suspendMapStyleRestoration = false;
      this.mapboxRendererSuspended = false;
      this.showMapboxRenderer();
      if (this.activeBaseMapMode === "satellite") {
        if (this.$map && this.$map.isStyleLoaded && this.$map.isStyleLoaded()) {
          this.applyThreeDimensionalMapSettings();
        }
        this.scheduleSatelliteTerrainRestore({
          hardRefresh: this.pendingSatelliteTerrainHardRefresh,
        });
      }
    },

    addThreeParkLayer() {
      this.enterStandaloneThreeMode();
    },

    removeThreeParkLayer() {
      this.exitStandaloneThreeMode();
    },

    rebuildThreeParkLayer() {
      if (this.shouldShowThreeParkLayer()) {
        this.enterStandaloneThreeMode();
      } else {
        this.exitStandaloneThreeMode();
      }
    },

    syncThreeParkLayer() {
      if (this.shouldShowThreeParkLayer()) {
        this.enterStandaloneThreeMode();
      } else {
        this.exitStandaloneThreeMode();
      }
    },

    async restoreMapOverlaysAfterStyleChange({ moveCamera = true } = {}) {
      if (!this.$map || this.suspendMapStyleRestoration) return;

      this.projectLayersReady = false;
      this.applyThreeDimensionalMapSettings();
      this.applyMapboxLabelLanguage(false);
      this.applyNeutralBaseRoadStyle();
      if (moveCamera) this.applyBaseMapCamera();

      await this.addProjectLayers();
      this.ensureDistrictLayers();
      this.setDistrictSourceData();
      this.renderMapData();
      this.applyNeutralBaseRoadStyle();
      if (this.isThreeDimensionalMapMode()) {
        this.applyThreeDimensionalMapSettings();
      }
      await this.syncConceptMapLayers();
      this.syncThreeParkLayer();

      // setStyle() can briefly report a style before all runtime layers/sources
      // are accepted. Try once more on the next idle frame so highlighted
      // project geometry stays visible after Street/Satellite/3D switches.
      if (!this.$map.getSource(PROJECT_SOURCE_ID)) {
        this.$map.once("idle", () => {
          this.restoreMapOverlaysAfterStyleChange({ moveCamera: false });
        });
      }
    },

    async setBaseMapMode(mode) {
      if (!this.baseMapModes.some((item) => item.value === mode)) return;
      if (mode === "3d" && !this.isProjectDetailsMap) return;

      const previousMode = this.activeBaseMapMode;
      const resetStateFromStandaloneThree =
        previousMode === "3d" && this.lastStandaloneThreeResetMapViewState
          ? { ...this.lastStandaloneThreeResetMapViewState }
          : null;
      const viewState =
        resetStateFromStandaloneThree || this.preserveCurrentViewState();
      if (resetStateFromStandaloneThree) {
        this.preservedMapViewState = { ...resetStateFromStandaloneThree };
      }

      // Pressing the already active button should not reset the user's camera.
      if (previousMode === mode) {
        if (mode === "3d" && !this.threeRenderer && !this.threeParkLoading) {
          await this.enterStandaloneThreeMode();
        }
        return;
      }

      if (mode === "3d") {
        this.clearMapCameraRestoreTimers();
        this.isBaseMapStyleSwitching = false;
        this.pendingBaseMapCameraState = null;
        this.activeBaseMapMode = "3d";
        if (viewState) this.preservedMapViewState = viewState;
        await this.enterStandaloneThreeMode();
        return;
      }

      this.activeBaseMapMode = mode;
      this.exitStandaloneThreeMode();
      if (previousMode === "3d") {
        this.lastStandaloneThreeResetMapViewState = null;
      }

      if (!this.$map) return;

      const cameraState = viewState || this.preservedMapViewState;
      if (cameraState) {
        this.lockCameraAcrossBaseMapStyleChange(cameraState);
      }

      this.suspendMapStyleRestoration = false;
      this.mapboxRendererSuspended = false;
      this.projectLayersReady = false;
      this.unbindProjectLayerEvents();
      this.$map.setStyle(this.getMapboxStyleForMode(mode));

      this.$map.once("style.load", () => {
        this.restoreMapCameraStateImmediately(cameraState);
      });

      this.$map.once("idle", async () => {
        await this.restoreMapOverlaysAfterStyleChange({
          moveCamera: false,
        });
        this.restoreMapCameraStateImmediately(cameraState);
        if (this.isThreeDimensionalMapMode(mode)) {
          this.applyThreeDimensionalMapSettings();
          if (mode === "satellite") {
            this.scheduleSatelliteTerrainRestore({
              hardRefresh:
                previousMode === "3d" ||
                this.pendingSatelliteTerrainHardRefresh,
            });
          }
          window.setTimeout(() => {
            if (this.activeBaseMapMode === mode) {
              this.applyThreeDimensionalMapSettings();
            }
          }, 300);
        }
      });
    },

    applyBaseMapCamera(duration = 0) {
      if (this.isStandaloneThreeMode) return;
      if (!this.$map) return;

      // Style changes must not force a preset pitch/bearing. Restore the exact
      // camera state captured before the user changed view modes.
      if (this.preservedMapViewState) {
        this.applyMapboxViewState(this.preservedMapViewState, duration);
      }
    },

    removeCustom3DBuildingsLayer() {
      if (!this.$map) return;

      [
        MAPBOX_3D_BUILDINGS_LAYER_ID,
        MAPBOX_3D_BUILDINGS_SAMPLING_LAYER_ID,
      ].forEach((layerId) => {
        try {
          if (this.$map.getLayer(layerId)) {
            this.$map.removeLayer(layerId);
          }
        } catch (error) {
          console.warn("3D building layer could not be removed:", error);
        }
      });
    },

    setMapboxStandard3DOptions() {
      if (!this.$map || typeof this.$map.setConfigProperty !== "function") {
        return;
      }

      const options = [
        ["basemap", "show3dObjects", true],
        ["basemap", "showBuildings", true],
        ["basemap", "showLandmarkIcons", true],
        ["basemap", "showRoadLabels", true],
        ["basemap", "showPlaceLabels", true],
        ["basemap", "showPointOfInterestLabels", true],
        ["basemap", "showTransitLabels", true],
        ["basemap", "lightPreset", "day"],
      ];

      options.forEach(([fragmentId, propertyName, value]) => {
        try {
          this.$map.setConfigProperty(fragmentId, propertyName, value);
        } catch (error) {
          // Some Mapbox styles do not support every Standard basemap option.
        }
      });
    },

    getMapboxBuildingVectorSourceId() {
      if (!this.$map || typeof this.$map.getStyle !== "function") return null;

      const style = this.$map.getStyle();
      const sources = style && style.sources ? style.sources : {};
      const preferredSourceIds = ["composite", "mapbox"];

      const preferredSourceId = preferredSourceIds.find(
        (sourceId) => sources[sourceId] && sources[sourceId].type === "vector",
      );

      if (preferredSourceId) return preferredSourceId;

      return (
        Object.keys(sources).find(
          (sourceId) =>
            sources[sourceId] && sources[sourceId].type === "vector",
        ) || null
      );
    },

    ensureStandaloneBuildingSource() {
      if (!this.$map || !this.$map.isStyleLoaded()) return;

      try {
        if (!this.$map.getSource(MAPBOX_3D_BUILDINGS_SOURCE_ID)) {
          this.$map.addSource(MAPBOX_3D_BUILDINGS_SOURCE_ID, {
            type: "vector",
            url: "mapbox://mapbox.mapbox-streets-v8",
          });
        }

        if (!this.$map.getLayer(MAPBOX_3D_BUILDINGS_SAMPLING_LAYER_ID)) {
          const samplingLayer = {
            id: MAPBOX_3D_BUILDINGS_SAMPLING_LAYER_ID,
            type: "fill",
            source: MAPBOX_3D_BUILDINGS_SOURCE_ID,
            "source-layer": "building",
            minzoom: MAPBOX_3D_BUILDINGS_MIN_ZOOM,
            paint: {
              "fill-color": "#ffffff",
              "fill-opacity": 0,
            },
          };

          try {
            this.$map.addLayer({ ...samplingLayer, slot: "top" });
          } catch (slotError) {
            this.$map.addLayer(samplingLayer);
          }
        }
      } catch (error) {
        console.warn("3D building sampling source could not be added:", error);
      }
    },

    addCustom3DBuildingsLayer() {
      if (!this.$map || !this.$map.isStyleLoaded()) return;
      if (this.$map.getLayer(MAPBOX_3D_BUILDINGS_LAYER_ID)) return;

      try {
        if (!this.$map.getSource(MAPBOX_3D_BUILDINGS_SOURCE_ID)) {
          this.$map.addSource(MAPBOX_3D_BUILDINGS_SOURCE_ID, {
            type: "vector",
            url: "mapbox://mapbox.mapbox-streets-v8",
          });
        }

        const layerDefinition = {
          id: MAPBOX_3D_BUILDINGS_LAYER_ID,
          type: "fill-extrusion",
          source: MAPBOX_3D_BUILDINGS_SOURCE_ID,
          "source-layer": "building",
          minzoom: MAPBOX_3D_BUILDINGS_MIN_ZOOM,
          paint: {
            "fill-extrusion-color": [
              "case",
              ["has", "height"],
              "#d4d8df",
              "#c9ced6",
            ],
            "fill-extrusion-height": [
              "interpolate",
              ["linear"],
              ["zoom"],
              MAPBOX_3D_BUILDINGS_MIN_ZOOM,
              0,
              MAPBOX_3D_BUILDINGS_MIN_ZOOM + 0.8,
              [
                "max",
                MAPBOX_3D_BUILDING_MIN_HEIGHT,
                [
                  "*",
                  MAPBOX_3D_BUILDING_HEIGHT_EXAGGERATION,
                  [
                    "coalesce",
                    ["to-number", ["get", "height"]],
                    ["to-number", ["get", "render_height"]],
                    ["*", 3.5, ["to-number", ["get", "levels"]]],
                    10,
                  ],
                ],
              ],
            ],
            "fill-extrusion-base": [
              "*",
              MAPBOX_3D_BUILDING_HEIGHT_EXAGGERATION,
              [
                "coalesce",
                ["to-number", ["get", "min_height"]],
                ["to-number", ["get", "render_min_height"]],
                0,
              ],
            ],
            "fill-extrusion-opacity": 0.86,
            "fill-extrusion-vertical-gradient": true,
          },
        };

        // In Mapbox Standard/Standard Satellite, putting the custom layer in the
        // "top" slot keeps it above the satellite terrain. In older styles the
        // slot property is harmless; if unsupported, retry without it.
        try {
          this.$map.addLayer({ ...layerDefinition, slot: "top" });
        } catch (slotError) {
          this.$map.addLayer(layerDefinition);
        }
      } catch (error) {
        console.warn("3D building layer could not be added:", error);
      }
    },

    applyThreeDimensionalMapSettings() {
      if (!this.$map || !this.$map.isStyleLoaded()) return;

      // ProjectDetails 3D is rendered by the standalone Three.js canvas.
      // Keep Mapbox in a lightweight/non-3D state for Street/Satellite only.
      if (this.activeBaseMapMode === "3d") {
        return;
      }

      if (!this.isThreeDimensionalMapMode()) {
        this.removeCustom3DBuildingsLayer();
        this.removeThreeParkLayer();
        try {
          this.$map.setTerrain(null);
        } catch (error) {
          // Some styles may not have terrain active yet.
        }
        return;
      }

      try {
        if (!this.$map.getSource(MAPBOX_TERRAIN_SOURCE_ID)) {
          this.$map.addSource(MAPBOX_TERRAIN_SOURCE_ID, {
            type: "raster-dem",
            url: "mapbox://mapbox.mapbox-terrain-dem-v1",
            tileSize: 512,
            maxzoom: 14,
          });
        }

        this.$map.setTerrain({
          source: MAPBOX_TERRAIN_SOURCE_ID,
          // Same terrain exaggeration for Main page, ProjectDetails,
          // ProjectType, Search, and every project category.
          exaggeration: MAPBOX_3D_TERRAIN_EXAGGERATION,
        });

        // Keep native Standard-style 3D options on where available, but do not
        // rely on them. The custom building layer below is what guarantees
        // buildings remain extruded in Satellite 3D mode.
        this.setMapboxStandard3DOptions();
        this.addCustom3DBuildingsLayer();

        // Style/source loading can briefly remove or hide custom layers. Recheck
        // on idle and add the building layer again if Mapbox dropped it.
        this.$map.once("idle", () => {
          if (this.isThreeDimensionalMapMode()) {
            this.addCustom3DBuildingsLayer();
          }
        });

        if (typeof this.$map.setMaxPitch === "function") {
          this.$map.setMaxPitch(85);
        }

        if (typeof this.$map.triggerRepaint === "function") {
          this.$map.triggerRepaint();
        }

        if (typeof this.$map.setFog === "function") {
          this.$map.setFog({
            color: "rgb(245, 246, 250)",
            "high-color": "rgb(205, 218, 235)",
            "horizon-blend": 0.18,
            "space-color": "rgb(230, 234, 240)",
            "star-intensity": 0.0,
          });
        }
      } catch (error) {
        console.warn("3D map settings could not be applied:", error);
      }
    },

    zoomMapIn() {
      if (this.isStandaloneThreeMode) {
        this.zoomStandaloneThree(0.82);
        return;
      }

      if (!this.$map) return;
      const currentZoom = Number(this.$map.getZoom() || this.zoom || 10);
      this.$map.easeTo({ zoom: Math.min(currentZoom + 1, 18), duration: 250 });
    },

    zoomMapOut() {
      if (this.isStandaloneThreeMode) {
        this.zoomStandaloneThree(1.22);
        return;
      }

      if (!this.$map) return;
      const currentZoom = Number(this.$map.getZoom() || this.zoom || 10);
      this.$map.easeTo({ zoom: Math.max(currentZoom - 1, 1), duration: 250 });
    },

    resetMapPositionFromControl() {
      if (this.isStandaloneThreeMode) {
        this.resetThreeParkCamera();
        return;
      }

      this.resetToDefaultMapView();
    },

    showLocationMessage(message) {
      this.locationErrorMessage = message;

      if (this.locationMessageTimeout) {
        window.clearTimeout(this.locationMessageTimeout);
      }

      this.locationMessageTimeout = window.setTimeout(() => {
        this.locationErrorMessage = "";
        this.locationMessageTimeout = null;
      }, 5000);
    },

    createUserLocationMarkerElement() {
      const element = document.createElement("div");
      element.className = "mapbox-user-location-marker";
      element.innerHTML = '<span class="mapbox-user-location-arrow"></span>';
      return element;
    },

    updateUserLocationMarker(position) {
      if (!this.$map || !position || !position.coords) return null;

      const lat = Number(position.coords.latitude);
      const lng = Number(position.coords.longitude);
      if (!Number.isFinite(lat) || !Number.isFinite(lng)) return null;

      if (
        Number.isFinite(position.coords.heading) &&
        position.coords.heading >= 0
      ) {
        this.userHeading = position.coords.heading;
      }

      const lngLat = [lng, lat];

      if (!this.userLocationMarker) {
        this.userLocationMarker = new this.mapboxgl.Marker({
          element: this.createUserLocationMarkerElement(),
          rotationAlignment: "map",
          pitchAlignment: "map",
        })
          .setLngLat(lngLat)
          .addTo(this.$map);
      } else {
        this.userLocationMarker.setLngLat(lngLat);
      }

      this.updateUserHeading(this.userHeading);
      this.hasUserLocation = true;
      this.isLocatingUser = false;
      return lngLat;
    },

    updateUserHeading(heading) {
      const normalizedHeading = Number(heading);
      if (!Number.isFinite(normalizedHeading)) return;

      this.userHeading = (normalizedHeading + 360) % 360;
      if (this.userLocationMarker && this.userLocationMarker.getElement) {
        const arrow = this.userLocationMarker
          .getElement()
          .querySelector(".mapbox-user-location-arrow");
        if (arrow) {
          arrow.style.transform = `rotate(${this.userHeading + 42}deg)`;
        }
      }
    },

    handleDeviceOrientation(event) {
      let heading = null;

      if (Number.isFinite(event.webkitCompassHeading)) {
        heading = event.webkitCompassHeading;
      } else if (Number.isFinite(event.alpha)) {
        const screenAngle = Number(
          window.screen?.orientation?.angle ?? window.orientation ?? 0,
        );
        heading = 360 - event.alpha + screenAngle;
      }

      this.updateUserHeading(heading);
    },

    async startDeviceOrientationTracking() {
      if (this.deviceOrientationHandler || typeof window === "undefined")
        return;
      if (!("DeviceOrientationEvent" in window)) return;

      try {
        if (
          typeof window.DeviceOrientationEvent.requestPermission === "function"
        ) {
          const permission =
            await window.DeviceOrientationEvent.requestPermission();
          if (permission !== "granted") return;
        }

        this.deviceOrientationHandler = (event) =>
          this.handleDeviceOrientation(event);
        window.addEventListener(
          "deviceorientationabsolute",
          this.deviceOrientationHandler,
          true,
        );
        window.addEventListener(
          "deviceorientation",
          this.deviceOrientationHandler,
          true,
        );
      } catch (error) {
        console.warn("Device direction permission was not granted:", error);
      }
    },

    startUserLocationWatch() {
      if (this.userLocationWatchId != null || !navigator.geolocation) return;

      this.userLocationWatchId = navigator.geolocation.watchPosition(
        (position) => this.updateUserLocationMarker(position),
        (error) => {
          this.isLocatingUser = false;
          console.warn("Unable to update current location:", error);
        },
        {
          enableHighAccuracy: true,
          maximumAge: 3000,
          timeout: 15000,
        },
      );
    },

    resetToDefaultMapView() {
      if (this.isStandaloneThreeMode) {
        this.resetThreeParkCamera();
        return;
      }

      if (!this.$map) return;

      this.isUserLocationViewActive = false;
      this.isLocatingUser = false;

      // On ProjectDetails pages the map's default view is the fitted geometry
      // for that one project, not the main overview center/zoom. Keep reset
      // scoped to the current page/view so it does not jump back to Main.
      if (this.activeViewMode === "project" && this.currentProjectId) {
        if (this.isConceptMapMode()) {
          this.applyConceptMapCamera(300);
        } else {
          this.fitMapToProjectIds([this.currentProjectId], 14, {
            forceStreetCamera: true,
          });
        }
        return;
      }

      // Search pages also have their own fitted default view.
      if (
        this.hasActiveSearchFilter() &&
        Array.isArray(this.activeSearchProjectIds) &&
        this.activeSearchProjectIds.length
      ) {
        this.fitMapToProjectIds(this.activeSearchProjectIds, 13, {
          forceStreetCamera: true,
        });
        return;
      }

      // Near-project views should reset to the currently focused project area.
      if (this.activeViewMode === "near" && this.nearProjectId) {
        this.fitMapToProjectIds([this.nearProjectId], 13, {
          forceStreetCamera: true,
        });
        return;
      }

      // Main page reset should return to the same camera angle as Street view
      // for every map mode. Keeping pitch/bearing at zero also prevents the
      // hidden 3D sampling map from leaving Satellite terrain in a flat state.
      this.$map.easeTo({
        center: this.getDefaultCenter(),
        zoom: Number(this.zoom || 10),
        pitch: 0,
        bearing: 0,
        duration: 300,
      });

      window.setTimeout(() => {
        if (this.isThreeDimensionalMapMode()) {
          this.applyThreeDimensionalMapSettings();
        }
      }, 360);
    },

    async goToUserLocation() {
      if (!this.$map) return;

      if (this.isUserLocationViewActive) {
        this.resetToDefaultMapView();
        return;
      }

      if (!navigator.geolocation) {
        this.showLocationMessage(
          this.isArabicUI()
            ? "المتصفح لا يدعم تحديد الموقع."
            : "Location services are not supported by this browser.",
        );
        return;
      }

      this.isLocatingUser = true;
      this.locationErrorMessage = "";
      this.startDeviceOrientationTracking();

      navigator.geolocation.getCurrentPosition(
        (position) => {
          const lngLat = this.updateUserLocationMarker(position);
          if (!lngLat) return;

          this.$map.easeTo({
            center: lngLat,
            zoom: this.isThreeDimensionalMapMode()
              ? MAPBOX_3D_BUILDINGS_CAMERA_ZOOM
              : 15,
            pitch: this.isThreeDimensionalMapMode()
              ? MAPBOX_3D_TERRAIN_PITCH
              : this.$map.getPitch(),
            bearing: this.isThreeDimensionalMapMode()
              ? MAPBOX_3D_TERRAIN_BEARING
              : this.$map.getBearing(),
            duration: 350,
          });
          this.isUserLocationViewActive = true;
          this.startUserLocationWatch();
        },
        (error) => {
          this.isLocatingUser = false;

          let message = this.isArabicUI()
            ? "تعذر الوصول إلى موقعك الحالي."
            : "Unable to access your current location.";

          if (error && error.code === 1) {
            message = this.isArabicUI()
              ? "يرجى السماح للموقع بالوصول إلى موقعك."
              : "Please allow this site to access your location.";
          }

          this.showLocationMessage(message);
        },
        {
          enableHighAccuracy: true,
          maximumAge: 0,
          timeout: 15000,
        },
      );
    },

    stopUserLocationTracking() {
      if (this.userLocationWatchId != null && navigator.geolocation) {
        navigator.geolocation.clearWatch(this.userLocationWatchId);
        this.userLocationWatchId = null;
      }

      if (this.deviceOrientationHandler) {
        window.removeEventListener(
          "deviceorientationabsolute",
          this.deviceOrientationHandler,
          true,
        );
        window.removeEventListener(
          "deviceorientation",
          this.deviceOrientationHandler,
          true,
        );
        this.deviceOrientationHandler = null;
      }

      if (this.userLocationMarker) {
        this.userLocationMarker.remove();
        this.userLocationMarker = null;
      }
    },

    highlightSearchProject(projectId) {
      if (!this.$map || !projectId) return;
      this.highlightedProjectId = this.normalizeProjectId(projectId);
      this.renderMapData();
    },

    clearSearchProjectHighlight() {
      this.highlightedProjectId = null;
      this.hoveredFeatureKey = null;
      this.renderMapData();
    },

    normalizeProgressValue(value) {
      const num = Number(String(value ?? "").replace(/[^\d.]/g, ""));
      return Number.isFinite(num) ? num : 0;
    },

    mapUiText(key, fallbackAr = "", fallbackEn = "") {
      const isArabic = this.isArabicUI();
      const fallback = isArabic
        ? fallbackAr || fallbackEn || key
        : fallbackEn || key;
      const translate =
        typeof this.$t === "function"
          ? this.$t
          : typeof this.$root?.$t === "function"
            ? this.$root.$t
            : null;
      const translateContext =
        typeof this.$t === "function" ? this : this.$root || this;

      if (translate) {
        try {
          const translated = translate.call(translateContext, key);
          if (translated && translated !== key) return translated;
        } catch (error) {
          // Fall back to the local label if the i18n plugin is unavailable here.
        }
      }

      return fallback;
    },

    mapUiLabel(item) {
      if (!item) return "";
      return this.mapUiText(
        item.labelKey || item.labelEn || item.value,
        item.labelAr || item.labelEn || item.value,
        item.labelEn || item.labelAr || item.value,
      );
    },

    isThreeDimensionalMapMode(mode = this.activeBaseMapMode) {
      // Satellite is the full Mapbox 3D environment: satellite imagery,
      // terrain, atmosphere and extruded buildings. The ProjectDetails-only
      // 3D mode remains the isolated standalone Three.js park cutout.
      return mode === "satellite" || mode === "3d";
    },

    isConceptMapMode(mode = this.activeBaseMapMode) {
      return mode === "3d-concept";
    },

    normalizeProjectText(value) {
      return String(value || "")
        .trim()
        .toLowerCase()
        .replace(/[\-_]+/g, " ")
        .replace(/\s+/g, " ");
    },

    parseImagePathList(value) {
      if (!value) return [];
      if (Array.isArray(value)) {
        return value
          .flatMap((item) => this.parseImagePathList(item))
          .map((item) => String(item || "").trim())
          .filter(Boolean);
      }

      if (typeof value === "string") {
        const trimmed = value.trim();
        if (!trimmed) return [];

        try {
          const parsed = JSON.parse(trimmed);
          if (Array.isArray(parsed)) return this.parseImagePathList(parsed);
        } catch (error) {
          // Keep parsing as a comma-separated string below.
        }

        return trimmed
          .replace(/[\r\n;|]+/g, ",")
          .split(",")
          .map((item) => item.trim().replace(/^['"]|['"]$/g, ""))
          .filter(Boolean);
      }

      return [];
    },

    isQadisiyah3ProjectItem(item = {}) {
      const id = this.normalizeProjectId(item.id);
      const projectId = this.normalizeProjectText(item.project_id);
      const nameEn = this.normalizeProjectText(item.name_en);
      const nameAr = String(item.name_ar || "").trim();

      return (
        id === QADISIYAH_3_PROJECT_ID ||
        projectId === this.normalizeProjectText(QADISIYAH_3_PROJECT_CODE) ||
        nameEn === QADISIYAH_3_PROJECT_NAME ||
        nameAr === "حديقة القادسية 3"
      );
    },

    getProjectItemById(projectId) {
      const id = this.normalizeProjectId(projectId);
      const rows = Array.isArray(this.$root?.data_src)
        ? this.$root.data_src
        : [];
      return (
        rows.find((item) => this.normalizeProjectId(item.id) === id) || null
      );
    },

    parseProjectGisValue(value) {
      if (!value) return [];

      if (Array.isArray(value)) {
        return value.flatMap((item) => this.parseProjectGisValue(item));
      }

      if (typeof value === "object") {
        return Object.keys(value).flatMap((key) => {
          const item = value[key];
          if (Array.isArray(item)) {
            return item.flatMap((entry) => {
              if (typeof entry === "string") {
                return [{ type: key, url: entry }];
              }
              if (entry && typeof entry === "object") {
                return [
                  {
                    type: entry.type || entry.layer || key,
                    url: entry.url || entry.path || entry.file || "",
                  },
                ];
              }
              return [];
            });
          }

          if (typeof item === "string") {
            return [{ type: key, url: item }];
          }

          if (item && typeof item === "object") {
            return [
              {
                type: item.type || item.layer || key,
                url: item.url || item.path || item.file || "",
              },
            ];
          }

          return [];
        });
      }

      const textValue = String(value || "").trim();
      if (!textValue) return [];

      if (
        (textValue.startsWith("{") && textValue.endsWith("}")) ||
        (textValue.startsWith("[") && textValue.endsWith("]"))
      ) {
        try {
          return this.parseProjectGisValue(JSON.parse(textValue));
        } catch (error) {
          // Continue with normal path parsing below.
        }
      }

      return textValue
        .replace(/[\r\n;|]+/g, ",")
        .split(",")
        .map((item) => item.trim().replace(/^['"]|['"]$/g, ""))
        .filter(Boolean)
        .map((url) => ({ type: "", url }));
    },

    normalizeStandaloneLayerFilter(value) {
      const text = String(value || "")
        .trim()
        .toLowerCase()
        .replace(/[\s_-]+/g, "");

      if (["vegetation", "plants", "planting"].includes(text)) {
        return "vegetation";
      }
      if (["paths", "path", "roads", "road", "paving"].includes(text)) {
        return "paths";
      }
      if (["water", "waterbody", "waters"].includes(text)) {
        return "water";
      }
      if (
        [
          "structures",
          "structure",
          "buildings",
          "building",
          "furniture",
        ].includes(text)
      ) {
        return "structures";
      }
      if (
        [
          "tse",
          "tseprimary",
          "tseprimarymains",
          "utility",
          "utilities",
          "lighting",
          "electricmanhole",
          "electricalmanhole",
          "electricpole",
          "electricalpole",
          "utilitypole",
          "powerline",
          "powerlines",
        ].includes(text)
      ) {
        return "tse";
      }
      return "";
    },

    getStandaloneThreeSubLayerOptions(layerKey) {
      const available = Array.isArray(this.threeSubLayerAvailability[layerKey])
        ? this.threeSubLayerAvailability[layerKey]
        : [];
      const visibility = this.threeSubLayerVisibility[layerKey] || {};

      return available.map((item) => ({
        key: item.key,
        label: item.label,
        active: visibility[item.key] !== false,
      }));
    },

    isStandaloneThreeParentLayerActive(layerKey) {
      if (!this.threeLayerVisibility[layerKey]) return false;
      const children = this.getStandaloneThreeSubLayerOptions(layerKey);
      return !children.length || children.some((child) => child.active);
    },

    isStandaloneThreeParentLayerPartiallyActive(layerKey) {
      if (!this.threeLayerVisibility[layerKey]) return false;
      const children = this.getStandaloneThreeSubLayerOptions(layerKey);
      if (children.length < 2) return false;
      const activeCount = children.filter((child) => child.active).length;
      return activeCount > 0 && activeCount < children.length;
    },

    getStandaloneFeatureSubLayerSource(feature) {
      const properties = (feature && feature.properties) || {};
      const lookup = new Map(
        Object.entries(properties).map(([key, value]) => [
          String(key).toLowerCase(),
          value,
        ]),
      );
      const candidates = [
        "sourcelayer",
        "source_layer",
        "_source_layer",
        "layername",
        "layer_name",
        "dataset",
        "dataset_name",
        "featureclass",
        "feature_class",
      ];

      for (const key of candidates) {
        const value = lookup.get(key);
        if (value == null) continue;
        const label = String(value)
          .replace(/\.(?:geojson|json|shp)$/i, "")
          .trim();
        if (label) return label;
      }

      return "";
    },

    getStandaloneUtilitySubLayerLabel(feature) {
      const properties = (feature && feature.properties) || {};
      const source = this.getStandaloneFeatureSubLayerSource(feature);
      const featureType = this.getStandaloneFeatureType(feature);
      const descriptor = [
        source,
        featureType,
        properties.category,
        properties.subtype,
        properties.Type,
        properties.type,
        properties.Remarks,
        properties.QC_REMARKS,
        properties.DC_REMARK,
        properties.APP_MODEL,
        properties.APP_3D_MODEL,
        properties.app_3d_model,
        properties["App 3D Model"],
        properties.APP_DISPLAY_LABEL,
        properties["App Display Label"],
        properties.APP_LABEL,
      ]
        .filter((value) => value != null && String(value).trim())
        .join(" ")
        .toLowerCase()
        .replace(/[_-]+/g, " ")
        .replace(/\s+/g, " ")
        .trim();

      if (!descriptor) return "";
      if (
        /\b(lighting|light\s*(?:post|pole|fixture)|luminaire|lamp)\b/.test(
          descriptor,
        )
      ) {
        return "Lighting";
      }
      if (/\b(?:electric(?:al)?\s*)?manhole(?:s)?\b/.test(descriptor)) {
        return "Electric Manhole";
      }
      if (/\b(?:electric(?:al)?|utility)\s*pole(?:s)?\b/.test(descriptor)) {
        return "Electric Pole";
      }
      if (/\bpower\s*line(?:s)?\b|\bpowerline(?:s)?\b/.test(descriptor)) {
        return "Powerline";
      }
      return "";
    },

    formatStandaloneSubLayerLabel(value, fallback = "Layer") {
      const cleaned = String(value || "")
        .replace(/\.(?:geojson|json|shp)$/i, "")
        .replace(/[_-]+/g, " ")
        .replace(/\s+/g, " ")
        .trim();
      if (!cleaned) return fallback;

      return cleaned
        .split(" ")
        .map((word) => {
          if (/^tse$/i.test(word)) return "TSE";
          if (/^hv$/i.test(word)) return "HV";
          if (/^lv$/i.test(word)) return "LV";
          return word.charAt(0).toUpperCase() + word.slice(1).toLowerCase();
        })
        .join(" ");
    },

    getStandaloneSubLayerDefinition(layerKey, feature) {
      const properties = (feature && feature.properties) || {};
      const source = this.getStandaloneFeatureSubLayerSource(feature);
      const featureType = this.getStandaloneFeatureType(feature);
      const geometryType = String(
        feature && feature.geometry && feature.geometry.type,
      ).toLowerCase();
      const descriptor = [
        source,
        featureType,
        properties.CL_LAND_COVER_DETAILED,
        properties.CL_LANDUSE_DETAILED,
        properties.CL_ROAD_TYPE,
        properties.ROAD_TYPE,
        properties.category,
        properties.subtype,
        properties.Type,
        properties.type,
        properties.Remarks,
        properties.QC_REMARKS,
        properties.DC_REMARK,
        properties.APP_MODEL,
        properties.APP_3D_MODEL,
        properties.app_3d_model,
        properties["App 3D Model"],
        properties.APP_DISPLAY_LABEL,
        properties["App Display Label"],
        properties.APP_LABEL,
      ]
        .filter((value) => value != null && String(value).trim())
        .join(" ")
        .toLowerCase()
        .replace(/[_-]+/g, " ");

      let label = "";

      if (layerKey === "vegetation") {
        if (
          /\b(ground\s*cover|groundcover|lawn|turf|grass)\b/.test(descriptor)
        ) {
          label = "GroundCover";
        } else if (/\b(shrub|shrubs|bush|bushes)\b/.test(descriptor)) {
          label = "Shrubs";
        } else if (
          /\b(tree|trees|palm|palms)\b/.test(descriptor) ||
          geometryType === "point" ||
          geometryType === "multipoint"
        ) {
          label = "Trees";
        } else {
          label = this.formatStandaloneSubLayerLabel(
            source || featureType,
            "Other Vegetation",
          );
        }
      } else if (layerKey === "paths") {
        if (
          /\b(car\s*parking|carpark|parking(?:\s+(?:lot|area))?)\b/.test(
            descriptor,
          )
        ) {
          label = "Car Parking";
        } else if (
          /\b(road|roadway|carriageway|street|vehicular|driveway|asphalt)\b/.test(
            descriptor,
          )
        ) {
          label = "Roads";
        } else if (/\b(sidewalk|footpath|foot\s*path)\b/.test(descriptor)) {
          label = "Sidewalk";
        } else if (/\b(stair|stairs|staircase|ramp)\b/.test(descriptor)) {
          label = "Stairs And Ramps";
        } else if (/\b(bridge|footbridge)\b/.test(descriptor)) {
          label = "Bridges";
        } else if (/\b(cycle\s*path|cycleway|bicycle)\b/.test(descriptor)) {
          label = "Cycle Paths";
        } else if (
          /\b(pedestrian|walkway|walk\s*way|walking|trail|path)\b/.test(
            descriptor,
          )
        ) {
          label = "Pedestrian Space";
        } else {
          label = this.formatStandaloneSubLayerLabel(
            source || featureType,
            "Other Paths",
          );
        }
      } else if (layerKey === "water") {
        if (/\bwater\s*feature\b|\bwaterfeature\b/.test(descriptor)) {
          label = "Water Features";
        } else if (
          /\b(jumping\s*fountain|fountain|water\s*jet)\b/.test(descriptor)
        ) {
          label = "Fountains";
        } else if (/\b(pool|swimming)\b/.test(descriptor)) {
          label = "Pools";
        } else if (/\b(channel|canal|stream|river)\b/.test(descriptor)) {
          label = "Channels";
        } else if (/\b(pond|lake)\b/.test(descriptor)) {
          label = "Ponds";
        } else {
          label = this.formatStandaloneSubLayerLabel(
            source || featureType,
            "Water Features",
          );
        }
      } else if (layerKey === "structures") {
        const sourceIsGeneric =
          !source ||
          Boolean(this.normalizeStandaloneLayerFilter(source)) ||
          /^(?:land\s*cover|landcover)$/i.test(source);
        if (!sourceIsGeneric) {
          label = this.formatStandaloneSubLayerLabel(
            source,
            "Other Structures",
          );
        } else if (
          /\b(lighting|light\s*(?:post|pole|fixture)|luminaire|lamp)\b/.test(
            descriptor,
          )
        ) {
          label = "Lighting";
        } else if (/\b(electrical\s*pole|utility\s*pole)\b/.test(descriptor)) {
          label = "Electrical Poles";
        } else if (/\b(manhole)\b/.test(descriptor)) {
          label = "Electrical Manholes";
        } else if (/\b(power\s*line|powerline)\b/.test(descriptor)) {
          label = "Powerlines";
        } else if (/\b(covered\s*pergola|pergola)\b/.test(descriptor)) {
          label = "Covered Pergolas";
        } else if (
          /\b(sport|court|playground|outdoor\s*gym|petanque)\b/.test(descriptor)
        ) {
          label = "Sports And Recreation";
        } else if (/\b(bench|bin|rack|bollard|furniture)\b/.test(descriptor)) {
          label = "Street Furniture";
        } else if (
          /\b(building|restaurant|toilet|bathroom|kiosk|prayer|office|facility)\b/.test(
            descriptor,
          )
        ) {
          label = "Buildings And Facilities";
        } else {
          label = this.formatStandaloneSubLayerLabel(
            featureType,
            "Other Structures",
          );
        }
      } else if (layerKey === "tse") {
        const utilityLabel = this.getStandaloneUtilitySubLayerLabel(feature);
        if (utilityLabel) label = utilityLabel;
        else if (/\bsecondary\b/.test(descriptor)) label = "TSE Secondary";
        else if (/\bprimary\b/.test(descriptor)) label = "TSE Primary";
        else
          label = this.formatStandaloneSubLayerLabel(
            source || featureType,
            "TSE Network",
          );
      } else {
        label = this.formatStandaloneSubLayerLabel(
          source || featureType,
          "Layer",
        );
      }

      const key =
        String(label || "layer")
          .trim()
          .toLowerCase()
          .replace(/[^a-z0-9]+/g, "-")
          .replace(/^-+|-+$/g, "") || "layer";

      return { key, label };
    },

    getStandaloneSubLayerSortIndex(layerKey, label) {
      const orders = {
        vegetation: ["Trees", "Shrubs", "GroundCover", "Other Vegetation"],
        paths: [
          "Pedestrian Space",
          "Sidewalk",
          "Cycle Paths",
          "Stairs And Ramps",
          "Bridges",
          "Roads",
          "Car Parking",
          "Other Paths",
        ],
        water: ["Fountains", "Pools", "Channels", "Ponds", "Water Features"],
        tse: [
          "Lighting",
          "Powerline",
          "Electric Manhole",
          "TSE Primary",
          "Electric Pole",
          "TSE Secondary",
          "TSE Network",
        ],
      };
      const order = orders[layerKey] || [];
      const index = order.indexOf(label);
      return index >= 0 ? index : order.length + 100;
    },

    clearStandaloneThreeSubLayers() {
      ["paths", "water", "structures", "vegetation", "tse"].forEach(
        (layerKey) => {
          this.$set(this.threeSubLayerAvailability, layerKey, []);
          this.$set(this.threeSubLayerVisibility, layerKey, {});
          this.$set(this.threeLayerExpanded, layerKey, false);
        },
      );
      this.$set(this, "threeStructureLabelVisibility", {});
      this.$set(this.threeToolState, "structureLabels", false);
      this.$set(this.threeToolState, "treeTypes", false);
      this.threeVisibleStructureLabels = [];
      this.threeLabelOverlaySignature = "";
    },

    updateStandaloneThreeSubLayerAvailability(geojson) {
      const layerKeys = ["paths", "water", "structures", "vegetation", "tse"];
      const buckets = new Map(layerKeys.map((key) => [key, new Map()]));
      const fallback = this.normalizeStandaloneLayerFilter(
        this.threeProjectGisFallbackLayer,
      );
      const features =
        geojson && Array.isArray(geojson.features) ? geojson.features : [];

      features.forEach((feature) => {
        const explicit = this.getStandaloneFeatureLayer(feature);
        const layerKey = explicit || fallback;
        if (!buckets.has(layerKey)) return;
        const definition = this.getStandaloneSubLayerDefinition(
          layerKey,
          feature,
        );
        buckets.get(layerKey).set(definition.key, definition);
      });

      layerKeys.forEach((layerKey) => {
        const definitions = Array.from(buckets.get(layerKey).values()).sort(
          (first, second) =>
            this.getStandaloneSubLayerSortIndex(layerKey, first.label) -
              this.getStandaloneSubLayerSortIndex(layerKey, second.label) ||
            first.label.localeCompare(second.label),
        );
        const previousVisibility = this.threeSubLayerVisibility[layerKey] || {};
        const nextVisibility = {};
        definitions.forEach((definition) => {
          nextVisibility[definition.key] =
            previousVisibility[definition.key] !== false;
        });
        this.$set(this.threeSubLayerAvailability, layerKey, definitions);
        this.$set(this.threeSubLayerVisibility, layerKey, nextVisibility);

        if (layerKey === "structures") {
          const previousLabelVisibility =
            this.threeStructureLabelVisibility || {};
          const nextLabelVisibility = {};
          definitions.forEach((definition) => {
            nextLabelVisibility[definition.key] =
              previousLabelVisibility[definition.key] === true;
          });
          this.$set(this, "threeStructureLabelVisibility", nextLabelVisibility);
          this.$set(
            this.threeToolState,
            "structureLabels",
            Object.values(nextLabelVisibility).some(Boolean),
          );
        }

        if (!definitions.length) {
          this.$set(this.threeLayerExpanded, layerKey, false);
        }
      });
    },

    filterStandaloneGeoJsonBySubLayer(geojson, layerKey, subLayerKey) {
      return {
        type: "FeatureCollection",
        properties: (geojson && geojson.properties) || {},
        features:
          geojson && Array.isArray(geojson.features)
            ? geojson.features.filter((feature) => {
                const definition = this.getStandaloneSubLayerDefinition(
                  layerKey,
                  feature,
                );
                return definition.key === subLayerKey;
              })
            : [],
      };
    },

    applyStandalonePathSubLayerStacking(group, definition = {}) {
      if (!group) return;

      const subLayerKey = String(definition.key || "").toLowerCase();
      const subLayerLabel = String(definition.label || "").toLowerCase();
      const isSidewalk =
        subLayerKey === "sidewalk" || subLayerLabel === "sidewalk";
      if (!isSidewalk) return;

      // Keep Sidewalk physically beneath every other PATHS sub-layer. This is
      // more reliable than child insertion order alone because opaque Three.js
      // objects can be re-sorted by material and camera depth before drawing.
      group.position.y -= THREE_PATH_SIDEWALK_STACK_DROP_METERS;
      group.userData.pathStackPriority = 0;

      group.traverse((object) => {
        if (!object || object === group) return;

        const isLineObject = Boolean(
          object.isLine || object.isLineSegments || object.isLineLoop,
        );
        if (object.isMesh || isLineObject || object.isPoints) {
          // All other path fills/outlines use render orders 6/7 or higher.
          object.renderOrder = isLineObject ? 5 : 4;
        }

        if (!object.isMesh || !object.material) return;
        const materials = Array.isArray(object.material)
          ? object.material
          : [object.material];

        materials.forEach((material) => {
          if (!material || !material.isMaterial) return;
          // Other path surfaces retain their negative polygon offset, so they
          // win the depth test wherever they overlap the lowered Sidewalk.
          material.polygonOffset = true;
          material.polygonOffsetFactor = 0;
          material.polygonOffsetUnits = 0;
          material.needsUpdate = true;
        });
      });
    },

    async createStandaloneSubLayeredGenericFeatureGroup(
      layerKey,
      geojson,
      modelData,
      contextGeoJson = geojson,
    ) {
      const parent = new THREE.Group();
      parent.name = `standalone-${layerKey}`;
      parent.userData.hasSubLayers = true;
      const definitions = this.getStandaloneThreeSubLayerOptions(layerKey);
      const activeDefinitions = definitions.length
        ? definitions
        : [
            {
              key: `${layerKey}-all`,
              label: this.formatStandaloneSubLayerLabel(layerKey),
            },
          ];
      let featureCount = 0;
      let polygonCount = 0;
      let halfBasketballCourtCount = 0;
      let fullBasketballCourtCount = 0;
      let mugaCourtCount = 0;
      let skateparkCount = 0;
      let detailedModularBuildingCount = 0;
      let pointCount = 0;
      let segmentCount = 0;

      for (const definition of activeDefinitions) {
        const childGeoJson = definitions.length
          ? this.filterStandaloneGeoJsonBySubLayer(
              geojson,
              layerKey,
              definition.key,
            )
          : geojson;
        if (!childGeoJson.features.length) continue;

        const child = await this.createStandaloneGenericFeatureGroup(
          layerKey,
          childGeoJson,
          modelData,
          contextGeoJson,
        );
        child.name = `standalone-${layerKey}-${definition.key}`;
        child.userData.subLayerKey = definition.key;
        child.userData.subLayerLabel = definition.label;
        if (layerKey === "paths") {
          this.applyStandalonePathSubLayerStacking(child, definition);
        }
        child.visible =
          !definitions.length ||
          (this.threeSubLayerVisibility[layerKey] || {})[definition.key] !==
            false;
        parent.add(child);
        featureCount += child.userData.featureCount || 0;
        polygonCount += child.userData.polygonCount || 0;
        halfBasketballCourtCount +=
          child.userData.halfBasketballCourtCount || 0;
        fullBasketballCourtCount +=
          child.userData.fullBasketballCourtCount || 0;
        mugaCourtCount += child.userData.mugaCourtCount || 0;
        skateparkCount += child.userData.skateparkCount || 0;
        detailedModularBuildingCount +=
          child.userData.detailedModularBuildingCount || 0;
        pointCount += child.userData.pointCount || 0;
        segmentCount += child.userData.segmentCount || 0;
      }

      parent.userData.featureCount = featureCount;
      parent.userData.polygonCount = polygonCount;
      parent.userData.halfBasketballCourtCount = halfBasketballCourtCount;
      parent.userData.fullBasketballCourtCount = fullBasketballCourtCount;
      parent.userData.mugaCourtCount = mugaCourtCount;
      parent.userData.skateparkCount = skateparkCount;
      parent.userData.detailedModularBuildingCount =
        detailedModularBuildingCount;
      parent.userData.pointCount = pointCount;
      parent.userData.segmentCount = segmentCount;
      return parent;
    },

    async createStandaloneSubLayeredVegetationGroup(
      geojson,
      modelData,
      loadToken,
    ) {
      const parent = new THREE.Group();
      parent.name = "standalone-vegetation";
      parent.userData.hasSubLayers = true;
      const definitions = this.getStandaloneThreeSubLayerOptions("vegetation");
      const activeDefinitions = definitions.length
        ? definitions
        : [{ key: "vegetation-all", label: "Vegetation" }];
      let treeCount = 0;
      let speciesCount = 0;
      let bucketCount = 0;

      for (const definition of activeDefinitions) {
        const childGeoJson = definitions.length
          ? this.filterStandaloneGeoJsonBySubLayer(
              geojson,
              "vegetation",
              definition.key,
            )
          : geojson;
        if (!childGeoJson.features.length) continue;

        const child = await this.createStandaloneVegetationGroup(
          childGeoJson,
          modelData,
          loadToken,
        );
        child.name = `standalone-vegetation-${definition.key}`;
        child.userData.subLayerKey = definition.key;
        child.userData.subLayerLabel = definition.label;
        child.visible =
          !definitions.length ||
          (this.threeSubLayerVisibility.vegetation || {})[definition.key] !==
            false;
        parent.add(child);
        treeCount += child.userData.treeCount || 0;
        speciesCount += child.userData.speciesCount || 0;
        bucketCount += child.userData.bucketCount || 0;
      }

      parent.userData.treeCount = treeCount;
      parent.userData.speciesCount = speciesCount;
      parent.userData.bucketCount = bucketCount;
      return parent;
    },

    getStandaloneStructureLabelDefinitions() {
      return Array.isArray(this.threeSubLayerAvailability.structures)
        ? this.threeSubLayerAvailability.structures
        : [];
    },

    isStandaloneStructureSubLayerLabelActive(subLayerKey) {
      return Boolean(
        subLayerKey &&
        this.threeStructureLabelVisibility &&
        this.threeStructureLabelVisibility[subLayerKey] === true,
      );
    },

    isStandaloneStructureLabelParentActive() {
      const definitions = this.getStandaloneStructureLabelDefinitions();
      if (!definitions.length) return false;
      return definitions.every((definition) =>
        this.isStandaloneStructureSubLayerLabelActive(definition.key),
      );
    },

    isStandaloneStructureLabelParentPartiallyActive() {
      const definitions = this.getStandaloneStructureLabelDefinitions();
      if (definitions.length < 2) return false;
      const activeCount = definitions.filter((definition) =>
        this.isStandaloneStructureSubLayerLabelActive(definition.key),
      ).length;
      return activeCount > 0 && activeCount < definitions.length;
    },

    setStandaloneAllStructureLabelVisibility(enabled) {
      const definitions = this.getStandaloneStructureLabelDefinitions();
      const nextVisibility = {
        ...(this.threeStructureLabelVisibility || {}),
      };
      definitions.forEach((definition) => {
        nextVisibility[definition.key] = Boolean(enabled);
      });

      this.$set(this, "threeStructureLabelVisibility", nextVisibility);
      const anyEnabled = definitions.some(
        (definition) => nextVisibility[definition.key] === true,
      );
      this.$set(this.threeToolState, "structureLabels", anyEnabled);

      if (!anyEnabled) {
        this.threeVisibleStructureLabels = [];
        this.threeLabelOverlaySignature = "";
      } else {
        this.updateStandaloneStructureLabelOverlay(true);
      }
    },

    handleStandaloneStructureLabelParentCheckboxChange(event) {
      const enabled = Boolean(event && event.target && event.target.checked);
      this.setStandaloneAllStructureLabelVisibility(enabled);
    },

    handleStandaloneStructureSubLayerLabelCheckboxChange(child, event) {
      if (!child || !child.key) return;
      const enabled = Boolean(event && event.target && event.target.checked);
      const nextVisibility = {
        ...(this.threeStructureLabelVisibility || {}),
        [child.key]: enabled,
      };
      this.$set(this, "threeStructureLabelVisibility", nextVisibility);

      const anyEnabled = Object.values(nextVisibility).some(Boolean);
      this.$set(this.threeToolState, "structureLabels", anyEnabled);

      if (!anyEnabled) {
        this.threeVisibleStructureLabels = [];
        this.threeLabelOverlaySignature = "";
      } else {
        this.updateStandaloneStructureLabelOverlay(true);
      }
    },

    setStandaloneAllSubLayerVisibility(layerKey, enabled) {
      const visibility = { ...(this.threeSubLayerVisibility[layerKey] || {}) };
      const definitions = Array.isArray(
        this.threeSubLayerAvailability[layerKey],
      )
        ? this.threeSubLayerAvailability[layerKey]
        : [];
      definitions.forEach((definition) => {
        visibility[definition.key] = Boolean(enabled);
      });
      this.$set(this.threeSubLayerVisibility, layerKey, visibility);
    },

    applyStandaloneSubLayerVisibility(layerKey) {
      const parent = this.threeLayerGroups && this.threeLayerGroups[layerKey];
      if (!parent) return;
      const parentVisible = Boolean(this.threeLayerVisibility[layerKey]);
      const visibility = this.threeSubLayerVisibility[layerKey] || {};

      parent.visible = parentVisible;
      parent.children.forEach((child) => {
        const subLayerKey =
          child && child.userData && child.userData.subLayerKey;
        if (!subLayerKey) return;
        child.visible = parentVisible && visibility[subLayerKey] !== false;
      });

      if (layerKey === "paths") {
        this.applyStandaloneConceptPathMode(this.threeToolState.conceptPaths);
      }
      if (layerKey === "structures") {
        this.updateStandaloneStructureLabelOverlay(true);
      }
    },

    toggleStandaloneThreeLayerExpanded(layerKey) {
      if (!this.threeLayerAvailability[layerKey]) return;
      this.$set(
        this.threeLayerExpanded,
        layerKey,
        !Boolean(this.threeLayerExpanded[layerKey]),
      );
    },

    async handleStandaloneThreeSubLayerCheckboxChange(layerKey, child, event) {
      if (!child || !this.threeLayerAvailability[layerKey]) return;
      const enabled = Boolean(event && event.target && event.target.checked);

      if (enabled && !this.hasStandaloneProjectLandscapeData()) {
        const visibility = {
          ...(this.threeSubLayerVisibility[layerKey] || {}),
          [child.key]: false,
        };
        this.$set(this.threeSubLayerVisibility, layerKey, visibility);
        this.$set(this.threeLayerVisibility, layerKey, false);
        this.threeLayerError =
          this.getStandaloneMissingLandscapeMessage(layerKey);
        return;
      }

      const visibility = { ...(this.threeSubLayerVisibility[layerKey] || {}) };
      visibility[child.key] = enabled;
      this.$set(this.threeSubLayerVisibility, layerKey, visibility);

      if (enabled) {
        if (!this.threeLayerVisibility[layerKey]) {
          await this.setStandaloneThreeLayerVisibility(layerKey, true, {
            preserveSubLayers: true,
          });
        } else {
          this.applyStandaloneSubLayerVisibility(layerKey);
        }
      } else {
        const anyEnabled = Object.values(visibility).some(Boolean);
        if (!anyEnabled) {
          this.$set(this.threeLayerVisibility, layerKey, false);
        }
        this.applyStandaloneSubLayerVisibility(layerKey);
      }
    },

    inferStandaloneThreeLayerType(typeValue, urlValue) {
      const direct = this.normalizeStandaloneLayerFilter(typeValue);
      if (direct) return direct;

      const text = `${String(typeValue || "")} ${String(urlValue || "")}`
        .trim()
        .toLowerCase()
        .replace(/[_-]+/g, " ");

      if (
        /\b(tree|trees|shrub|shrubs|groundcover|ground[_ -]?cover|vegetation|plant|plants)\b/.test(
          text,
        )
      ) {
        return "vegetation";
      }

      if (
        /\b(path|paths|walkway|walkways|trail|trails|cycle|road|parking|pedestrian)\b/.test(
          text,
        )
      ) {
        return "paths";
      }

      if (/\b(water|pond|lake|stream|river|fountain|canal)\b/.test(text)) {
        return "water";
      }

      if (
        /\b(tse|treated sewage effluent|utility|utilities|lighting|light\s*(?:post|pole|fixture)|luminaire|lamp|electric(?:al)?\s*manhole|electric(?:al)?\s*pole|utility\s*pole|power\s*line|powerline)\b/.test(
          text,
        )
      ) {
        return "tse";
      }

      if (
        /\b(structure|structures|building|buildings|facility|facilities|pavilion|kiosk|furniture)\b/.test(
          text,
        )
      ) {
        return "structures";
      }

      return "";
    },

    getStandaloneProjectGisUrl(projectItem = null) {
      const item =
        projectItem ||
        (this.currentProjectId
          ? this.getProjectItemById(this.currentProjectId)
          : null);
      if (!item) return "";

      const entries = this.parseProjectGisValue(item.gis);
      const first = entries.find((entry) =>
        Boolean(String(entry && entry.url ? entry.url : "").trim()),
      );
      return first ? String(first.url).trim() : "";
    },

    getStandaloneThreeLayerUrls(projectItem = null) {
      const item =
        projectItem ||
        (this.currentProjectId
          ? this.getProjectItemById(this.currentProjectId)
          : null);
      const output = {
        combined: "",
        paths: "",
        water: "",
        structures: "",
        vegetation: "",
        tse: "",
      };
      if (!item) return output;

      const entries = this.parseProjectGisValue(item.gis);
      entries.forEach((entry) => {
        const url = String(entry && entry.url ? entry.url : "").trim();
        if (!url) return;

        const layerType = this.inferStandaloneThreeLayerType(
          entry && entry.type,
          url,
        );
        if (layerType && !output[layerType]) {
          output[layerType] = url;
        } else if (!layerType && !output.combined) {
          output.combined = url;
        }
      });

      // A single combined park FeatureCollection is used by every discovered
      // filter. Availability is determined from each feature's layerFilter
      // property after the file is loaded.
      if (output.combined) {
        ["paths", "water", "structures", "vegetation", "tse"].forEach((key) => {
          if (!output[key] && this.threeLayerAvailability[key]) {
            output[key] = output.combined;
          }
        });
      }

      return output;
    },

    getStandaloneFeatureLayer(feature) {
      // Electrical assets and lighting belong to the expandable UTILITY
      // filter even when the source GeoJSON originally grouped them under
      // STRUCTURES. TSE remains in the same internal group for compatibility.
      if (this.getStandaloneUtilitySubLayerLabel(feature)) return "tse";

      const properties = (feature && feature.properties) || {};
      const explicitLayer = this.normalizeStandaloneLayerFilter(
        properties.layerFilter ||
          properties.layer_filter ||
          properties.appLayer ||
          properties.app_layer ||
          properties.category,
      );
      if (explicitLayer) return explicitLayer;

      // Combined park files can preserve the original exported filename in
      // _source_layer instead of carrying an explicit layerFilter property.
      // Infer the app layer so those combined files remain immediately usable.
      const source = this.getStandaloneFeatureSubLayerSource(feature)
        .trim()
        .toLowerCase()
        .replace(/[\s_-]+/g, "");

      if (
        ["trees", "tree", "shrubs", "shrub", "groundcover"].includes(source)
      ) {
        return "vegetation";
      }
      if (["roads", "road", "paths", "path"].includes(source)) {
        return "paths";
      }
      if (
        [
          "tseprimary",
          "tseprimarymains",
          "tse",
          "electricpole",
          "electricalpole",
          "manhole",
          "electricmanhole",
          "electricalmanhole",
          "lighting",
          "powerline",
          "powerlines",
        ].includes(source)
      ) {
        return "tse";
      }
      if (
        ["furniture", "structures", "structure", "buildings"].includes(source)
      ) {
        return "structures";
      }
      if (["water", "waters"].includes(source)) return "water";

      if (source === "landcover") {
        const descriptor = [
          properties.QC_REMARKS,
          properties.DC_REMARK,
          properties.APP_MODEL,
          properties.APP_LABEL,
          properties.CL_LAND_COVER_DETAILED,
          properties.CL_LANDUSE_DETAILED,
          properties.Type,
          properties.type,
        ]
          .filter((value) => value != null && String(value).trim())
          .join(" ")
          .toLowerCase()
          .replace(/[_-]+/g, " ");

        if (
          /\bwater\s*feature\b|\bwaterfeature\b|fountain|pond|pool|lake|channel|canal\b/.test(
            descriptor,
          )
        ) {
          return "water";
        }
        if (
          /\bpedestrian\s*space|sidewalk|foot\s*path|footpath|walkway|road|parking\b/.test(
            descriptor,
          )
        ) {
          return "paths";
        }

        // The remaining land-cover polygons in the ArcGIS export are built
        // facilities such as sports courts, pergolas, kiosks and buildings.
        return "structures";
      }

      return this.normalizeStandaloneLayerFilter(source);
    },

    filterStandaloneGeoJsonByLayer(geojson, layerKey) {
      const normalized = this.normalizeStandaloneLayerFilter(layerKey);
      const fallback = this.normalizeStandaloneLayerFilter(
        this.threeProjectGisFallbackLayer,
      );

      return {
        type: "FeatureCollection",
        properties: (geojson && geojson.properties) || {},
        features:
          geojson && Array.isArray(geojson.features)
            ? geojson.features.filter((feature) => {
                const explicitLayer = this.getStandaloneFeatureLayer(feature);
                return explicitLayer
                  ? explicitLayer === normalized
                  : Boolean(fallback && fallback === normalized);
              })
            : [],
      };
    },

    async prepareStandaloneProjectGis(projectItem = null) {
      const item =
        projectItem ||
        (this.currentProjectId
          ? this.getProjectItemById(this.currentProjectId)
          : null);
      const entries = item ? this.parseProjectGisValue(item.gis) : [];
      const first = entries.find((entry) =>
        Boolean(String(entry && entry.url ? entry.url : "").trim()),
      );
      const url = first ? String(first.url).trim() : "";
      const fallbackLayer = first
        ? this.inferStandaloneThreeLayerType(first.type, first.url)
        : "";

      this.threeProjectGisUrl = url;
      this.threeProjectGisData = null;
      this.threeProjectGisFallbackLayer = fallbackLayer;
      this.threeProjectGisLoadError = "";

      this.$set(this.threeLayerAvailability, "terrainAfter", true);

      // Prime the UI from the GIS column before fetching the file. This keeps
      // configured GIS filters looking clickable even when Landscape/LandXML is
      // missing, and also prevents a transient GIS fetch problem from greying
      // every filter out. Once the GeoJSON loads, the exact feature contents
      // below refine these optimistic values.
      const configuredAvailability = new Set();
      let hasCombinedGisEntry = false;
      entries.forEach((entry) => {
        const entryUrl = String(entry && entry.url ? entry.url : "").trim();
        if (!entryUrl) return;
        const inferred = this.inferStandaloneThreeLayerType(
          entry && entry.type,
          entryUrl,
        );
        if (inferred) configuredAvailability.add(inferred);
        else hasCombinedGisEntry = true;
      });
      if (hasCombinedGisEntry) {
        ["paths", "water", "structures", "vegetation", "tse"].forEach((key) =>
          configuredAvailability.add(key),
        );
      }
      ["paths", "water", "structures", "vegetation", "tse"].forEach((key) => {
        this.$set(
          this.threeLayerAvailability,
          key,
          configuredAvailability.has(key),
        );
      });
      this.clearStandaloneThreeSubLayers();

      if (!url) return null;

      let geojson = this.threeProjectGisCache.get(url);
      if (!geojson) {
        geojson = await this.loadStandaloneGeoJson(url);
        this.threeProjectGisCache.set(url, geojson);
      }

      this.threeProjectGisData = geojson;
      const available = new Set();
      const metadataLayers =
        geojson &&
        geojson.properties &&
        Array.isArray(geojson.properties.availableLayerFilters)
          ? geojson.properties.availableLayerFilters
          : [];

      metadataLayers.forEach((value) => {
        const key = this.normalizeStandaloneLayerFilter(value);
        if (key) available.add(key);
      });

      geojson.features.forEach((feature) => {
        const key = this.getStandaloneFeatureLayer(feature);
        if (key) available.add(key);
      });

      // Backwards compatibility for an older project whose gis value points
      // directly to trees.geojson, paths.geojson, etc. The new combined park
      // files should contain layerFilter on every feature and do not depend on
      // their filename.
      if (!available.size && fallbackLayer) {
        available.add(fallbackLayer);
      }

      ["paths", "water", "structures", "vegetation", "tse"].forEach((key) => {
        this.$set(this.threeLayerAvailability, key, available.has(key));
      });
      this.updateStandaloneThreeSubLayerAvailability(geojson);

      return geojson;
    },

    applyStandaloneSatelliteGroundDisplay(
      enabled = this.threeLayerVisibility.satelliteGround,
      texture = this.threeSatelliteGroundTexture,
      textureRepeat = { x: 1, y: 1 },
    ) {
      const useSatellite = Boolean(enabled && texture && texture.isTexture);
      const useSealedGround = Boolean(
        useSatellite &&
        this.threeLayerVisibility.compactedGround &&
        this.threeCompactedGroundTexture &&
        texture === this.threeCompactedGroundTexture,
      );
      const repeatX = Math.max(
        1,
        Number(textureRepeat && textureRepeat.x) || 1,
      );
      const repeatY = Math.max(
        1,
        Number(textureRepeat && textureRepeat.y) || 1,
      );
      const groups = this.threeLayerGroups || {};

      ["terrainBefore", "terrainAfter"].forEach((groupKey) => {
        const terrainGroup = groups[groupKey];
        if (
          !terrainGroup ||
          typeof terrainGroup.getObjectByName !== "function"
        ) {
          return;
        }

        const surfaceMesh = terrainGroup.getObjectByName("terrain-top-surface");
        const material = surfaceMesh && surfaceMesh.material;
        const uniforms = material && material.uniforms;
        if (!uniforms) return;

        if (
          texture &&
          texture.isTexture &&
          uniforms.uSatelliteMap &&
          uniforms.uSatelliteMap.value !== texture
        ) {
          const previousTexture = uniforms.uSatelliteMap.value;
          uniforms.uSatelliteMap.value = texture;

          // Each terrain group starts with its own 1x1 fallback texture. Once
          // the real satellite image is assigned that tiny fallback is no
          // longer needed.
          if (
            previousTexture &&
            previousTexture !== texture &&
            previousTexture.isDataTexture &&
            typeof previousTexture.dispose === "function"
          ) {
            previousTexture.dispose();
          }
        }

        if (uniforms.uUseSatellite) {
          uniforms.uUseSatellite.value = useSatellite ? 1 : 0;
        }
        if (uniforms.uGroundTextureRepeat) {
          uniforms.uGroundTextureRepeat.value.set(repeatX, repeatY);
        }
        if (uniforms.uSatelliteBrightness) {
          uniforms.uSatelliteBrightness.value = useSatellite ? 1.0 : 0.0;
        }
        if (uniforms.uGridSpacing) {
          const baseGridSpacing = Math.max(
            1,
            Number(surfaceMesh.userData.baseGridSpacing) ||
              Number(uniforms.uGridSpacing.value) ||
              4,
          );
          uniforms.uGridSpacing.value = useSealedGround
            ? THREE_SEALED_GROUND_GRID_CELL_METERS
            : baseGridSpacing;
        }
        if (uniforms.uGridIntensity) {
          // SEALED GROUND needs stronger, tighter cells because the photo
          // texture is visually busy. Satellite Ground stays subtler.
          uniforms.uGridIntensity.value = useSealedGround
            ? THREE_SEALED_GROUND_GRID_OPACITY
            : useSatellite
              ? 0.24
              : 0.18;
        }
      });
    },

    getStandaloneSatelliteGroundBounds() {
      const modelData = this.threeParkModelData;
      if (!modelData) return null;

      const beforeData = modelData.terrainBeforeData || modelData;
      return beforeData.geographicBounds || modelData.geographicBounds || null;
    },

    async setStandaloneSatelliteGroundVisibility(enabled) {
      const nextEnabled = Boolean(enabled);
      if (nextEnabled && !this.threeLayerVisibility.terrain) return;

      this.$set(this.threeLayerVisibility, "satelliteGround", nextEnabled);

      if (nextEnabled && this.threeLayerVisibility.compactedGround) {
        this.$set(this.threeLayerVisibility, "compactedGround", false);
        this.threeCompactedGroundLoadToken += 1;
      }

      if (!nextEnabled) {
        this.threeSatelliteGroundLoadToken += 1;
        if (this.threeLayerLoadingKey === "satelliteGround") {
          this.threeLayerLoadingKey = "";
        }
        this.applyStandaloneSatelliteGroundDisplay(false);
        return;
      }

      if (
        this.threeSatelliteGroundTexture &&
        this.threeSatelliteGroundTexture.isTexture
      ) {
        this.applyStandaloneSatelliteGroundDisplay(
          true,
          this.threeSatelliteGroundTexture,
        );
        return;
      }

      const bounds = this.getStandaloneSatelliteGroundBounds();
      if (!bounds) {
        this.$set(this.threeLayerVisibility, "satelliteGround", false);
        this.threeLayerError = this.mapUiText(
          "Satellite Ground is unavailable for this project.",
          "أرضية القمر الصناعي غير متاحة لهذا المشروع.",
          "Satellite Ground is unavailable for this project.",
        );
        return;
      }

      const loadToken = this.threeSatelliteGroundLoadToken + 1;
      this.threeSatelliteGroundLoadToken = loadToken;
      this.threeLayerLoadingKey = "satelliteGround";
      this.threeLayerError = "";

      try {
        const texture = await this.loadStandaloneSatelliteTexture(bounds);

        if (
          loadToken !== this.threeSatelliteGroundLoadToken ||
          !this.threeLayerVisibility.satelliteGround ||
          !this.threeScene
        ) {
          if (texture && typeof texture.dispose === "function") {
            texture.dispose();
          }
          return;
        }

        if (!texture) {
          throw new Error("Satellite texture unavailable.");
        }

        if (
          this.threeRenderer &&
          this.threeRenderer.capabilities &&
          typeof this.threeRenderer.capabilities.getMaxAnisotropy === "function"
        ) {
          texture.anisotropy = Math.min(
            8,
            this.threeRenderer.capabilities.getMaxAnisotropy(),
          );
          texture.needsUpdate = true;
        }

        this.threeSatelliteGroundTexture = texture;
        this.applyStandaloneSatelliteGroundDisplay(true, texture);
      } catch (error) {
        if (loadToken !== this.threeSatelliteGroundLoadToken) return;

        console.warn("Satellite Ground could not be loaded.", error);
        this.$set(this.threeLayerVisibility, "satelliteGround", false);
        this.applyStandaloneSatelliteGroundDisplay(false);
        this.threeLayerError = this.mapUiText(
          "Satellite Ground could not be loaded. Check the Mapbox connection and try again.",
          "تعذر تحميل أرضية القمر الصناعي. تحقق من اتصال Mapbox ثم حاول مرة أخرى.",
          "Satellite Ground could not be loaded. Check the Mapbox connection and try again.",
        );
      } finally {
        if (
          loadToken === this.threeSatelliteGroundLoadToken &&
          this.threeLayerLoadingKey === "satelliteGround"
        ) {
          this.threeLayerLoadingKey = "";
        }
      }
    },

    getStandaloneCompactedGroundTextureRepeat() {
      const modelData = this.threeParkModelData;
      const beforeData =
        modelData && (modelData.terrainBeforeData || modelData);
      const localBounds = beforeData && beforeData.localBounds;
      if (!localBounds) return { x: 1, y: 1 };

      const width = Math.max(
        1,
        Number(localBounds.maxX) - Number(localBounds.minX),
      );
      const depth = Math.max(
        1,
        Number(localBounds.maxZ) - Number(localBounds.minZ),
      );
      const tileMeters = Math.max(4, THREE_PARK_COMPACTED_GROUND_TILE_METERS);

      return {
        x: Math.max(1, Math.min(64, width / tileMeters)),
        y: Math.max(1, Math.min(64, depth / tileMeters)),
      };
    },

    async setStandaloneCompactedGroundVisibility(enabled) {
      const nextEnabled = Boolean(enabled);
      if (nextEnabled && !this.threeLayerVisibility.terrain) return;

      this.$set(this.threeLayerVisibility, "compactedGround", nextEnabled);

      if (!nextEnabled) {
        this.threeCompactedGroundLoadToken += 1;
        if (this.threeLayerLoadingKey === "compactedGround") {
          this.threeLayerLoadingKey = "";
        }

        if (this.threeLayerVisibility.satelliteGround) {
          this.applyStandaloneSatelliteGroundDisplay(
            true,
            this.threeSatelliteGroundTexture,
          );
        } else {
          this.applyStandaloneSatelliteGroundDisplay(false);
        }
        return;
      }

      // Satellite Ground and SEALED GROUND are alternate colour textures
      // for the same terrain surface, so only one can be active at a time.
      if (this.threeLayerVisibility.satelliteGround) {
        this.$set(this.threeLayerVisibility, "satelliteGround", false);
        this.threeSatelliteGroundLoadToken += 1;
      }

      const repeat = this.getStandaloneCompactedGroundTextureRepeat();
      if (
        this.threeCompactedGroundTexture &&
        this.threeCompactedGroundTexture.isTexture
      ) {
        this.applyStandaloneSatelliteGroundDisplay(
          true,
          this.threeCompactedGroundTexture,
          repeat,
        );
        return;
      }

      const loadToken = this.threeCompactedGroundLoadToken + 1;
      this.threeCompactedGroundLoadToken = loadToken;
      this.threeLayerLoadingKey = "compactedGround";
      this.threeLayerError = "";

      try {
        const texture = await this.loadStandaloneCompactedGroundTexture();

        if (
          loadToken !== this.threeCompactedGroundLoadToken ||
          !this.threeLayerVisibility.compactedGround ||
          !this.threeScene
        ) {
          if (texture && typeof texture.dispose === "function") {
            texture.dispose();
          }
          return;
        }

        if (!texture) {
          throw new Error("SEALED GROUND texture unavailable.");
        }

        if (
          this.threeRenderer &&
          this.threeRenderer.capabilities &&
          typeof this.threeRenderer.capabilities.getMaxAnisotropy === "function"
        ) {
          texture.anisotropy = Math.min(
            8,
            this.threeRenderer.capabilities.getMaxAnisotropy(),
          );
          texture.needsUpdate = true;
        }

        this.threeCompactedGroundTexture = texture;
        this.applyStandaloneSatelliteGroundDisplay(true, texture, repeat);
      } catch (error) {
        if (loadToken !== this.threeCompactedGroundLoadToken) return;

        console.warn("SEALED GROUND could not be loaded.", error);
        this.$set(this.threeLayerVisibility, "compactedGround", false);
        this.applyStandaloneSatelliteGroundDisplay(false);
        this.threeLayerError = this.mapUiText(
          "SEALED GROUND could not be loaded. Add sealed_ground.png to public/images.",
          "تعذر تحميل الأرضية المختومة. أضف sealed_ground.png إلى public/images.",
          "SEALED GROUND could not be loaded. Add sealed_ground.png to public/images.",
        );
      } finally {
        if (
          loadToken === this.threeCompactedGroundLoadToken &&
          this.threeLayerLoadingKey === "compactedGround"
        ) {
          this.threeLayerLoadingKey = "";
        }
      }
    },

    applyStandaloneTerrainDisplayMode(
      mode = this.threeTerrainMode,
      visible = this.threeLayerVisibility.terrain,
    ) {
      const groups = this.threeLayerGroups || {};
      const terrainContainer = groups.terrain;
      const terrainBeforeGroup = groups.terrainBefore;
      const terrainAfterGroup = groups.terrainAfter;
      if (!terrainContainer) return;

      const shouldShow = Boolean(visible);
      terrainContainer.visible = shouldShow;

      // Terrain (before) and Terrain (after) are separate Three.js groups.
      // "before" uses Mapbox existing-ground elevations; "after" uses the
      // Civil 3D LandXML TIN. Only the selected group is visible.
      if (terrainBeforeGroup) {
        terrainBeforeGroup.visible = shouldShow && mode === "before";
      }
      if (terrainAfterGroup) {
        terrainAfterGroup.visible = shouldShow && mode === "after";
      }
    },

    resetStandaloneThreeLayerFilters() {
      this.threeLayerLoadingKey = "";
      this.threeLayerError = "";
      this.threeTerrainMode = "before";
      this.threeLayerVisibility.terrain = true;
      this.threeLayerVisibility.satelliteGround = false;
      this.threeLayerVisibility.compactedGround = false;
      this.threeLayerVisibility.paths = false;
      this.threeLayerVisibility.water = false;
      this.threeLayerVisibility.structures = false;
      this.threeLayerVisibility.vegetation = false;
      this.threeLayerVisibility.tse = false;
      this.threeToolState.structureLabels = false;
      this.threeToolState.conceptPaths = false;
      this.threeToolState.treeTypes = false;
      ["paths", "water", "structures", "vegetation", "tse"].forEach((key) => {
        this.$set(this.threeLayerExpanded, key, false);
      });
      this.threeVisibleStructureLabels = [];
      this.threeVegetationLoadToken += 1;
      this.threeSatelliteGroundLoadToken += 1;
      this.threeCompactedGroundLoadToken += 1;

      if (this.threeLayerGroups) {
        this.applyStandaloneTerrainDisplayMode("before", true);
        this.applyStandaloneSatelliteGroundDisplay(false);

        ["paths", "water", "structures", "vegetation", "tse"].forEach((key) => {
          const group = this.threeLayerGroups[key];
          if (group) group.visible = false;
        });
      }
    },

    async handleStandaloneTreeTypeCheckboxChange(event) {
      const nextValue = Boolean(event && event.target && event.target.checked);
      this.$set(this.threeToolState, "treeTypes", nextValue);

      // This checkbox intentionally performs exactly the same action as the
      // former stacked-layers toolbar button: it only opens/closes the Tree
      // Type panel and loads the tree-type summary when opened.
      if (nextValue) {
        await this.ensureStandaloneTreeTypeStats();
      }
    },

    async handleStandaloneThreeToolButtonClick(tool) {
      if (!tool || !tool.enabled) return;

      if (tool.key === "layers") {
        const nextValue = !this.threeToolState.treeTypes;
        this.$set(this.threeToolState, "treeTypes", nextValue);
        if (nextValue) {
          await this.ensureStandaloneTreeTypeStats();
        }
        return;
      }

      if (tool.key === "measure") {
        const nextValue = !this.threeToolState.conceptPaths;
        this.$set(this.threeToolState, "conceptPaths", nextValue);
        if (nextValue && !this.threeLayerVisibility.paths) {
          await this.setStandaloneThreeLayerVisibility("paths", true);
        }
        this.applyStandaloneConceptPathMode(nextValue);
        return;
      }

      if (tool.key !== "structureLabels") return;

      const nextValue = !this.threeToolState.structureLabels;

      if (nextValue && !this.threeLayerVisibility.structures) {
        await this.setStandaloneThreeLayerVisibility("structures", true);
      }

      // Keep the existing toolbar button as the master "all structure labels"
      // control. The new Layer Filters column can then refine labels by
      // individual Structures sub-category.
      this.setStandaloneAllStructureLabelVisibility(nextValue);
    },

    getStandaloneTreeIconColor(speciesValue) {
      const palette = this.getStandaloneTreePalette(speciesValue);
      const value = Number(palette && palette.foliage);
      return `#${(Number.isFinite(value) ? value : 0x0f412c)
        .toString(16)
        .padStart(6, "0")}`;
    },

    buildStandaloneTreeTypeStats(geojson) {
      const features =
        geojson && Array.isArray(geojson.features) ? geojson.features : [];
      const counts = new Map();

      features.forEach((feature) => {
        const geometry = feature && feature.geometry;
        if (
          !geometry ||
          (geometry.type !== "Point" && geometry.type !== "MultiPoint")
        ) {
          return;
        }

        const coordinateCount =
          geometry.type === "MultiPoint" && Array.isArray(geometry.coordinates)
            ? geometry.coordinates.length
            : 1;
        if (!coordinateCount) return;

        const properties = (feature && feature.properties) || {};
        const rawSpecies = this.getStandaloneTreeSpecies(properties);
        const name =
          rawSpecies ||
          this.mapUiText(
            "Unknown tree type",
            "نوع شجرة غير معروف",
            "Unknown tree type",
          );
        const existing = counts.get(name) || {
          name,
          count: 0,
          archetype: this.getStandaloneTreeArchetype(rawSpecies),
          color: this.getStandaloneTreeIconColor(rawSpecies),
        };
        existing.count += coordinateCount;
        counts.set(name, existing);
      });

      return Array.from(counts.values()).sort((a, b) => {
        if (b.count !== a.count) return b.count - a.count;
        return String(a.name).localeCompare(String(b.name));
      });
    },

    async ensureStandaloneTreeTypeStats() {
      if (this.threeTreeTypeStats.length || this.threeTreeTypeLoading) return;
      const urls = this.getStandaloneThreeLayerUrls();
      const url = urls.vegetation || urls.combined;
      if (!url) return;

      this.threeTreeTypeLoading = true;
      try {
        const sourceGeoJson =
          this.threeProjectGisData &&
          this.threeProjectGisUrl === String(url || "")
            ? this.threeProjectGisData
            : await this.loadStandaloneGeoJson(url);
        const vegetationGeoJson = this.filterStandaloneGeoJsonByLayer(
          sourceGeoJson,
          "vegetation",
        );
        this.threeTreeTypeStats =
          this.buildStandaloneTreeTypeStats(vegetationGeoJson);
      } catch (error) {
        console.error("Tree type data could not be loaded:", error);
        this.threeTreeTypeStats = [];
      } finally {
        this.threeTreeTypeLoading = false;
      }
    },

    getStandaloneStructureLabelText(featureType = "", properties = {}) {
      const propertyEntries = Object.entries(properties || {});
      const propertyLookup = new Map(
        propertyEntries.map(([key, value]) => [
          String(key).toLowerCase(),
          value,
        ]),
      );
      const getProperty = (...keys) => {
        for (const key of keys) {
          const value = propertyLookup.get(String(key).toLowerCase());
          if (value == null) continue;
          const text = String(value).trim();
          if (!text || /^<?null>?$/i.test(text)) continue;
          return text;
        }
        return "";
      };

      const appLabel = getProperty(
        "APP_DISPLAY_LABEL",
        "App Display Label",
        "app_display_label",
        "APP_LABEL",
        "app_label",
      );
      const appModel = getProperty(
        "APP_MODEL",
        "app_model",
        "APP_3D_MODEL",
        "app_3d_model",
        "App 3D Model",
      );
      const appHoopSide = getProperty("APP_HOOP_SIDE", "app_hoop_side");
      const detailedLandCover = getProperty(
        "CL_LAND_COVER_DETAILED",
        "CL_LANDUSE_DETAILED",
      );
      const qcRemarks = getProperty("QC_REMARKS");
      const departmentLabel = getProperty(
        "LABEL",
        "LABEL_EN",
        "DISPLAY_LABEL",
        "FACILITY_NAME",
        "FACILITY_TYPE",
        "ASSET_NAME",
        "ASSET_TYPE",
      );
      const typeValue = getProperty("Type", "featureType", "feature_type");
      const remarks = getProperty("Remarks", "DC_REMARK");
      const sourceLayer = getProperty("sourceLayer");
      const searchableText =
        `${featureType || ""} ${appModel} ${appLabel} ${appHoopSide} ${departmentLabel} ${detailedLandCover} ${qcRemarks} ${typeValue} ${remarks} ${sourceLayer}`.toLowerCase();

      // Do not label repetitive lighting fixtures or light posts.
      if (
        /lighting|light[_ -]?(?:post|pole|fixture)|luminaire|lamp/.test(
          searchableText,
        )
      ) {
        return null;
      }

      // Keep the label view focused on facilities/buildings. Repetitive
      // furniture and utility assets otherwise consume the overlap slots and
      // hide Restaurant, Bathrooms, Sports Courts, etc.
      if (
        /bench|recycling[_ -]?bin|waste[_ -]?bin|manhole|bollard|bike[_ -]?rack|spherical[_ -]?concrete|electrical[_ -]?pole|power[_ -]?line/.test(
          searchableText,
        )
      ) {
        return null;
      }

      // App Display Label is an app-specific display override. Preserve its exact
      // capitalization, punctuation and units instead of replacing it with a
      // generated/fallback structure label. APP_LABEL remains supported for
      // older GeoJSON exports.
      //
      // Covered Pergola is intentionally authored in ALL CAPS in the Qadisiyah
      // GIS data. Normalize this one legacy label defensively as well, because
      // older combined GeoJSON exports may still contain "Covered Pergola".
      if (appLabel) {
        if (/^covered\s+pergola$/i.test(appLabel)) return "COVERED PERGOLA";
        return appLabel;
      }

      // Use the actual ArcGIS properties shown in the project data.
      if (/sport[_ -]?(?:facilit|court)/.test(searchableText)) {
        return "Sports Courts";
      }
      if (/restaurant/.test(searchableText)) return "Restaurant";
      if (
        /double[_ -]?toilet|simple[_ -]?toilet|toilet|bathroom|washroom|restroom/.test(
          searchableText,
        )
      ) {
        return "Bathrooms";
      }
      if (/prayer[_ -]?room|mosque/.test(searchableText)) return "Prayer Room";
      if (
        /cycling[_ -]?(?:center|centre)|cycle[_ -]?(?:center|centre)/.test(
          searchableText,
        )
      ) {
        return "Cycling Center";
      }
      if (/visitor[_ -]?(?:center|centre)/.test(searchableText)) {
        return "Visitor Centre";
      }
      if (/administration[_ -]?(?:office|building)/.test(searchableText)) {
        return "Administration Office";
      }
      if (/maintenance[_ -]?(?:and[_ -]?water[_ -]?tanks|water[_ -]?tanks)/.test(searchableText)) {
        return "Maintenance and Water Tanks";
      }
      if (/maintenance[_ -]?building/.test(searchableText)) {
        return "Maintenance Building";
      }
      if (/electrical[_ -]?room/.test(searchableText)) {
        return "Electrical Room";
      }
      if (/electrical[_ -]?building/.test(searchableText)) {
        return "Electrical Building";
      }
      if (/technical[_ -]?building/.test(searchableText)) {
        return "Technical Building";
      }
      if (/large[_ -]?kiosk/.test(searchableText)) return "Large Kiosk";
      if (/small[_ -]?kiosk/.test(searchableText)) return "Small Kiosk";
      if (/kiosk/.test(searchableText)) return "Kiosk";

      const preferredKeys = [
        "APP_DISPLAY_LABEL",
        "App Display Label",
        "app_display_label",
        "APP_LABEL",
        "app_label",
        "LABEL",
        "LABEL_EN",
        "DISPLAY_LABEL",
        "FACILITY_NAME",
        "FACILITY_TYPE",
        "ASSET_NAME",
        "ASSET_TYPE",
        "CL_LAND_COVER_DETAILED",
        "CL_LANDUSE_DETAILED",
        "DisplayName",
        "display_name",
        "Name",
        "name",
        "FacilityName",
        "facility_name",
        "Title",
        "title",
        "QC_REMARKS",
        "Type",
        "type",
        "Remarks",
        "featureType",
        "feature_type",
        "sourceLayer",
      ];
      let label = "";
      preferredKeys.some((key) => {
        const value = getProperty(key);
        if (!value) return false;
        label = value;
        return true;
      });
      if (!label) {
        label = String(featureType || "Structure").replace(/[_-]+/g, " ");
      }

      label = label
        .replace(/[_-]+/g, " ")
        .replace(/\btable tenis\b/gi, "Table Tennis")
        .replace(/\bbbq\b/gi, "BBQ")
        .replace(/\bel\b/gi, "Electrical")
        .replace(/\bwc\b/gi, "Bathrooms")
        .replace(/\s+/g, " ")
        .trim();

      if (!label) {
        return this.mapUiText("Structure", "منشأ", "Structure");
      }

      return label
        .toLowerCase()
        .replace(/\b\w/g, (match) => match.toUpperCase());
    },

    getStandaloneStructureLabelPriority(featureType = "", properties = {}) {
      const searchableText = `${featureType || ""} ${
        properties.APP_MODEL ||
        properties.app_model ||
        properties.APP_3D_MODEL ||
        properties.app_3d_model ||
        properties["App 3D Model"] ||
        ""
      } ${
        properties.APP_DISPLAY_LABEL ||
        properties["App Display Label"] ||
        properties.app_display_label ||
        properties.APP_LABEL ||
        properties.app_label ||
        ""
      } ${
        properties.QC_REMARKS || ""
      } ${properties.CL_LAND_COVER_DETAILED || ""} ${
        properties.Type || ""
      } ${properties.Remarks || ""}`.toLowerCase();

      // App Display Label is an intentional ArcGIS -> app display override, so labels
      // such as Water Tanks Building, Electrical Building, Maintenance Building,
      // and Themed Gardens Classrooms should win overlap placement.
      if (
        String(
          properties.APP_DISPLAY_LABEL ||
            properties["App Display Label"] ||
            properties.app_display_label ||
            properties.APP_LABEL ||
            properties.app_label ||
            "",
        ).trim()
      ) {
        return 0;
      }

      if (
        /sport[_ -]?(?:facilit|court)|restaurant|toilet|bathroom|washroom|restroom|prayer|cycling[_ -]?(?:center|centre)|visitor[_ -]?(?:center|centre)|administration|maintenance[_ -]?building|electrical[_ -]?building|technical[_ -]?building|kiosk/.test(
          searchableText,
        )
      ) {
        return 0;
      }
      if (
        /pergola|playground|outdoor[_ -]?gym|petanque|running[_ -]?track/.test(
          searchableText,
        )
      ) {
        return 1;
      }
      if (
        /bench|bin|rack|manhole|pole|spherical|fixture/.test(searchableText)
      ) {
        return 3;
      }
      return 2;
    },

    getStandaloneStructureAnnotationPosition(feature, modelData) {
      if (!feature || !feature.geometry || !modelData) return null;
      const geometry = feature.geometry;
      const collect = [];
      const pushCoordinate = (coordinate) => {
        if (
          Array.isArray(coordinate) &&
          Number.isFinite(Number(coordinate[0])) &&
          Number.isFinite(Number(coordinate[1]))
        ) {
          collect.push([Number(coordinate[0]), Number(coordinate[1])]);
        }
      };

      // For polygons, use only the exterior ring. Including holes and every
      // nested ring can pull a label away from the actual facility building.
      if (geometry.type === "Polygon") {
        ((geometry.coordinates && geometry.coordinates[0]) || []).forEach(
          pushCoordinate,
        );
      } else if (geometry.type === "MultiPolygon") {
        const largestOuter =
          (geometry.coordinates || [])
            .map((polygon) => (polygon && polygon[0]) || [])
            .sort((a, b) => b.length - a.length)[0] || [];
        largestOuter.forEach(pushCoordinate);
      } else {
        const walk = (value) => {
          if (!Array.isArray(value) || !value.length) return;
          if (typeof value[0] === "number") {
            pushCoordinate(value);
            return;
          }
          value.forEach(walk);
        };
        walk(geometry.coordinates);
      }
      if (!collect.length) return null;

      const minLongitude = Math.min(
        ...collect.map((coordinate) => coordinate[0]),
      );
      const maxLongitude = Math.max(
        ...collect.map((coordinate) => coordinate[0]),
      );
      const minLatitude = Math.min(
        ...collect.map((coordinate) => coordinate[1]),
      );
      const maxLatitude = Math.max(
        ...collect.map((coordinate) => coordinate[1]),
      );
      const longitude = (minLongitude + maxLongitude) / 2;
      const latitude = (minLatitude + maxLatitude) / 2;

      const local = this.lngLatToTerrainMeters(
        [longitude, latitude],
        modelData.origin,
      );
      return {
        local,
        geometryType: geometry.type,
      };
    },

    buildStandaloneBuildingAnnotations(modelData) {
      const data =
        modelData && modelData.terrainAfterData
          ? modelData.terrainAfterData
          : modelData;
      const footprints =
        data && Array.isArray(data.buildingFootprints)
          ? data.buildingFootprints
          : [];

      return footprints
        .map((building, index) => {
          const outer = Array.isArray(building.outer) ? building.outer : [];
          if (!outer.length) return null;
          const center = outer.reduce(
            (accumulator, point) => {
              accumulator.x += Number(point.x) || 0;
              accumulator.z += Number(point.z) || 0;
              return accumulator;
            },
            { x: 0, z: 0 },
          );
          center.x /= outer.length;
          center.z /= outer.length;

          const properties = building.properties || {};
          const rawLabel =
            building.label ||
            properties.name_en ||
            properties.name ||
            properties.Name ||
            properties.type ||
            properties.class ||
            "Building";
          const text = String(rawLabel)
            .replace(/[_-]+/g, " ")
            .replace(/\s+/g, " ")
            .trim()
            .replace(/\b\w/g, (match) => match.toUpperCase());

          return {
            id: `mapbox-building-${index}`,
            text: text || "Building",
            x: center.x,
            z: center.z,
            absoluteY: Number(building.topY) + 0.9,
            yOffset: 0,
            source: "building",
            subLayerKey: "buildings-and-facilities",
          };
        })
        .filter(Boolean);
    },

    buildStandaloneStructureAnnotations(geojson, modelData) {
      const features =
        geojson && Array.isArray(geojson.features) ? geojson.features : [];
      const annotations = [];

      features.forEach((feature, index) => {
        const properties = (feature && feature.properties) || {};
        const featureType = this.getStandaloneFeatureType(feature);
        const anchor = this.getStandaloneStructureAnnotationPosition(
          feature,
          modelData,
        );
        if (!anchor) return;

        // APP_LABEL is authored display text from ArcGIS. Always render it
        // exactly as stored (including ALL CAPS, punctuation and units) and
        // only use the automatic label formatter when APP_LABEL is absent.
        let authoredAppLabel = String(
          this.getStandaloneCaseInsensitiveProperty(properties, [
            "APP_LABEL",
            "app_label",
          ]) || "",
        ).trim();

        // Guarantee the pergola label matches the authored all-caps GIS label
        // even if an older deployed/combined GeoJSON contains Title Case.
        if (/^covered\s+pergola$/i.test(authoredAppLabel)) {
          authoredAppLabel = "COVERED PERGOLA";
        }

        const label =
          authoredAppLabel ||
          this.getStandaloneStructureLabelText(featureType, properties);
        if (!label) return;

        let yOffset = 1.35;
        if (this.isStandaloneHalfBasketballCourtStructure(properties)) {
          // Keep the app label above the 3.05 m rim and backboard.
          yOffset = 4.75;
        } else if (/polygon/i.test(anchor.geometryType || "")) {
          yOffset = Math.max(
            1.3,
            this.getStandaloneStructureHeight(featureType, properties) + 0.8,
          );
        } else if (/line/i.test(anchor.geometryType || "")) {
          yOffset = 1.5;
        } else if (/pole|light/.test(featureType)) {
          yOffset = Math.max(
            2.5,
            Math.min(
              8,
              this.getStandaloneNumericProperty(
                properties,
                ["PoleHeight", "Height", "height_m"],
                5.6,
              ) + 0.8,
            ),
          );
        } else if (/manhole/.test(featureType)) {
          yOffset = 0.9;
        }

        annotations.push({
          id: `${properties.OBJECTID || properties.objectid || index}-${featureType || "structure"}`,
          text: label,
          x: anchor.local.x,
          z: anchor.local.z,
          yOffset,
          source: "arcgis-structure",
          subLayerKey: this.getStandaloneSubLayerDefinition(
            "structures",
            feature,
          ).key,
          priority: this.getStandaloneStructureLabelPriority(
            featureType,
            properties,
          ),
        });
      });

      return annotations;
    },

    updateStandaloneStructureLabelOverlay(force = false) {
      if (
        !this.threeToolState.structureLabels ||
        !this.threeCamera ||
        !this.$refs.threeParkHost ||
        !this.threeLayerVisibility.structures
      ) {
        if (this.threeVisibleStructureLabels.length) {
          this.threeVisibleStructureLabels = [];
          this.threeLabelOverlaySignature = "";
        }
        return;
      }

      const now = Date.now();
      if (!force && now - (this.threeLabelOverlayUpdateAt || 0) < 110) {
        return;
      }
      this.threeLabelOverlayUpdateAt = now;

      const annotations = Array.isArray(this.threeStructureAnnotations)
        ? this.threeStructureAnnotations
        : [];
      if (!annotations.length) {
        if (this.threeVisibleStructureLabels.length) {
          this.threeVisibleStructureLabels = [];
          this.threeLabelOverlaySignature = "";
        }
        return;
      }

      const host = this.$refs.threeParkHost;
      const width = Math.max(1, host.clientWidth || 0);
      const height = Math.max(1, host.clientHeight || 0);
      const camera = this.threeCamera;
      const world = new THREE.Vector3();
      const projected = [];

      const structureSubLayerVisibility =
        this.threeSubLayerVisibility.structures || {};
      const structureLabelVisibility = this.threeStructureLabelVisibility || {};
      annotations.forEach((annotation) => {
        if (
          annotation.subLayerKey &&
          structureSubLayerVisibility[annotation.subLayerKey] === false
        ) {
          return;
        }
        if (
          annotation.subLayerKey &&
          structureLabelVisibility[annotation.subLayerKey] !== true
        ) {
          return;
        }
        const terrainY = this.getStandaloneTerrainHeightAtPoint(
          { x: annotation.x, z: annotation.z },
          this.threeParkModelData,
        );
        const worldY = Number.isFinite(Number(annotation.absoluteY))
          ? Number(annotation.absoluteY)
          : terrainY + Number(annotation.yOffset || 0);
        world.set(annotation.x, worldY, annotation.z);
        const point = world.clone().project(camera);
        if (point.z < -1 || point.z > 1) return;
        const screenX = (point.x * 0.5 + 0.5) * width;
        const screenY = (-point.y * 0.5 + 0.5) * height;
        if (
          screenX < 16 ||
          screenX > width - 16 ||
          screenY < 12 ||
          screenY > height - 12
        ) {
          return;
        }
        const focusTarget =
          this.threeControls && this.threeControls.target
            ? this.threeControls.target
            : new THREE.Vector3();
        const focusDistance = Math.hypot(
          world.x - focusTarget.x,
          world.z - focusTarget.z,
        );

        projected.push({
          id: annotation.id,
          text: annotation.text,
          x: Math.round(screenX),
          y: Math.round(screenY),
          depth: point.z,
          distance: camera.position.distanceTo(world),
          focusDistance,
          priority: Number.isFinite(Number(annotation.priority))
            ? Number(annotation.priority)
            : String(annotation.id || "").startsWith("mapbox-building-")
              ? 1
              : 2,
        });
      });

      projected.sort((a, b) => {
        return a.priority - b.priority || a.distance - b.distance;
      });

      const target =
        this.threeControls && this.threeControls.target
          ? this.threeControls.target
          : new THREE.Vector3();
      const cameraDistance = camera.position.distanceTo(target);
      const maxDimension = Math.max(
        1,
        Number(
          this.threeParkModelData && this.threeParkModelData.maxDimension,
        ) || 500,
      );

      // Only the local cluster around the camera/orbit target uses collision
      // avoidance. Labels outside that focus area remain exactly over their
      // structures, even when they overlap one another.
      //
      // The focus radius shrinks naturally as the camera gets closer, so zooming
      // into one group of facilities gives readable labels for that group while
      // preserving precise anchors everywhere else in the park.
      const focusRadius = THREE.MathUtils.clamp(
        cameraDistance * 0.32,
        maxDimension * 0.055,
        maxDimension * 0.2,
      );
      const focusLabels = [];
      const distantLabels = [];

      projected.forEach((annotation) => {
        if (Number(annotation.focusDistance) <= focusRadius) {
          focusLabels.push(annotation);
        } else {
          distantLabels.push(annotation);
        }
      });

      const occupiedRectangles = [];
      const visible = [];
      const rectanglesOverlap = (first, second, padding = 6) =>
        !(
          first.right + padding <= second.left ||
          first.left >= second.right + padding ||
          first.bottom + padding <= second.top ||
          first.top >= second.bottom + padding
        );
      const estimateLabelSize = (text) => ({
        width: Math.min(
          220,
          Math.max(72, String(text || "").length * 6.25 + 22),
        ),
        height: 24,
      });

      // First keep every distant label at the exact projected structure
      // position. They intentionally do not participate in collision detection.
      distantLabels.forEach((annotation) => {
        if (visible.length >= 240) return;
        visible.push({
          ...annotation,
          x: Math.round(annotation.x),
          y: Math.round(annotation.y),
        });
      });

      // For the local cluster only, search nearby screen positions and prevent
      // those labels from overlapping each other. Distant labels are ignored by
      // this collision system so they never get pushed away from their structures.
      const baseOffsets = [[0, 0]];
      const xSteps = [0, 88, -88, 176, -176, 264, -264, 352, -352];
      const ySteps = [0, -32, 32, -64, 64, -96, 96, -128, 128];

      xSteps.forEach((offsetX) => {
        ySteps.forEach((offsetY) => {
          if (offsetX === 0 && offsetY === 0) return;
          baseOffsets.push([offsetX, offsetY]);
        });
      });
      baseOffsets.sort(
        (first, second) =>
          first[0] * first[0] +
          first[1] * first[1] -
          (second[0] * second[0] + second[1] * second[1]),
      );

      focusLabels
        .sort((a, b) => {
          return (
            a.priority - b.priority ||
            a.focusDistance - b.focusDistance ||
            a.distance - b.distance
          );
        })
        .forEach((annotation) => {
          if (visible.length >= 240) return;
          const size = estimateLabelSize(annotation.text);
          let placed = null;

          baseOffsets.some(([offsetX, offsetY]) => {
            const labelX = annotation.x + offsetX;
            const labelY = annotation.y + offsetY;
            const rectangle = {
              left: labelX - size.width / 2,
              right: labelX + size.width / 2,
              top: labelY - 10 - size.height,
              bottom: labelY - 10,
            };

            if (
              rectangle.left < 8 ||
              rectangle.right > width - 8 ||
              rectangle.top < 8 ||
              rectangle.bottom > height - 8
            ) {
              return false;
            }

            if (
              occupiedRectangles.some((existing) =>
                rectanglesOverlap(rectangle, existing, 8),
              )
            ) {
              return false;
            }

            placed = {
              ...annotation,
              x: Math.round(labelX),
              y: Math.round(labelY),
            };
            occupiedRectangles.push(rectangle);
            return true;
          });

          // If a very dense local cluster exhausts every candidate position,
          // keep the label anchored rather than hiding it entirely.
          if (!placed) {
            placed = {
              ...annotation,
              x: Math.round(annotation.x),
              y: Math.round(annotation.y),
            };
          }

          visible.push(placed);
        });

      // Include the rendered text in the overlay signature. Previously the
      // signature only tracked id/x/y, so a label whose APP_LABEL changed only
      // in capitalization (for example "Covered Pergola" -> "COVERED PERGOLA")
      // could be treated as unchanged and Vue would keep the stale title-cased
      // text until another camera/position change forced an overlay refresh.
      const signature = visible
        .map(
          (item) => `${item.id}:${item.x}:${item.y}:${String(item.text || "")}`,
        )
        .join("|");
      if (!force && signature === this.threeLabelOverlaySignature) {
        return;
      }
      this.threeLabelOverlaySignature = signature;
      this.threeVisibleStructureLabels = visible;
    },

    handleStandaloneThreeLayerCheckboxClick(layer) {
      // All available layers, including Terrain, now use the checkbox change
      // handler. Keep this method for the existing template event binding.
      if (!layer) return;
    },

    handleStandaloneThreeLayerCheckboxChange(layer, event) {
      if (!layer) return;

      this.setStandaloneThreeLayerVisibility(
        layer.key,
        Boolean(event && event.target && event.target.checked),
      );
    },

    async setStandaloneThreeLayerVisibility(layerKey, enabled, options = {}) {
      if (layerKey === "satelliteGround") {
        await this.setStandaloneSatelliteGroundVisibility(enabled);
        return;
      }

      if (layerKey === "compactedGround") {
        await this.setStandaloneCompactedGroundVisibility(enabled);
        return;
      }

      if (layerKey === "terrainBefore") {
        const visible = Boolean(enabled);
        this.threeTerrainMode = "before";
        this.$set(this.threeLayerVisibility, "terrain", visible);
        this.applyStandaloneTerrainDisplayMode("before", visible);
        this.threeLayerError = "";

        if (!visible && this.threeLayerVisibility.satelliteGround) {
          await this.setStandaloneSatelliteGroundVisibility(false);
        }
        if (!visible && this.threeLayerVisibility.compactedGround) {
          await this.setStandaloneCompactedGroundVisibility(false);
        }
        return;
      }

      if (layerKey === "terrainAfter") {
        const visible = Boolean(enabled);
        this.threeTerrainMode = "after";
        this.$set(this.threeLayerVisibility, "terrain", visible);
        this.applyStandaloneTerrainDisplayMode("after", visible);

        // Missing Landscape/LandXML is represented by the open perimeter shell:
        // keep the checkbox usable, but explain why the Terrain (after) top
        // surface is intentionally absent.
        this.threeLayerError =
          visible && !this.hasStandaloneProjectLandscapeData()
            ? this.getStandaloneMissingLandscapeMessage("terrainAfter")
            : "";

        if (!visible && this.threeLayerVisibility.satelliteGround) {
          await this.setStandaloneSatelliteGroundVisibility(false);
        }
        if (!visible && this.threeLayerVisibility.compactedGround) {
          await this.setStandaloneCompactedGroundVisibility(false);
        }
        return;
      }

      if (!this.threeLayerAvailability[layerKey]) return;

      // The GIS column controls whether these filters are available in the UI,
      // while Landscape/LandXML controls whether they can be placed on the
      // proposed Terrain (after) surface. With GIS but no landscape, allow the
      // checkbox interaction, then return a precise layer-level error instead
      // of failing the entire 3D terrain view.
      if (enabled && !this.hasStandaloneProjectLandscapeData()) {
        this.$set(this.threeLayerVisibility, layerKey, false);
        const existingGroup =
          this.threeLayerGroups && this.threeLayerGroups[layerKey];
        if (existingGroup) existingGroup.visible = false;
        this.threeLayerError =
          this.getStandaloneMissingLandscapeMessage(layerKey);
        return;
      }

      const preserveSubLayers = Boolean(options && options.preserveSubLayers);
      if (!preserveSubLayers) {
        this.setStandaloneAllSubLayerVisibility(layerKey, Boolean(enabled));
      }

      const urls = this.getStandaloneThreeLayerUrls();
      const url = urls[layerKey] || urls.combined;
      if (!url) return;

      this.threeLayerError = "";
      this.$set(this.threeLayerVisibility, layerKey, Boolean(enabled));

      const existingGroup =
        this.threeLayerGroups && this.threeLayerGroups[layerKey];

      if (existingGroup) {
        this.makeStandaloneLayerGroupOpaque(existingGroup);
        existingGroup.visible = Boolean(enabled);
        this.applyStandaloneSubLayerVisibility(layerKey);
        if (layerKey === "paths") {
          this.applyStandaloneConceptPathMode(this.threeToolState.conceptPaths);
        }
        if (layerKey === "structures" && !enabled) {
          this.threeVisibleStructureLabels = [];
        }
        if (layerKey === "structures" && enabled) {
          this.updateStandaloneStructureLabelOverlay(true);
        }
        return;
      }

      if (!enabled) {
        if (layerKey === "structures") {
          this.threeVisibleStructureLabels = [];
        }
        return;
      }

      if (layerKey === "vegetation") {
        await this.ensureStandaloneVegetationLayer(url);
        return;
      }

      await this.ensureStandaloneGenericFeatureLayer(layerKey, url);
    },

    async loadStandaloneGeoJson(url) {
      const response = await fetch(String(url || ""), {
        cache: "no-store",
        credentials: "same-origin",
      });

      if (!response.ok) {
        throw new Error(
          `Unable to load GIS layer (${response.status} ${response.statusText}).`,
        );
      }

      const data = await response.json();
      if (
        !data ||
        data.type !== "FeatureCollection" ||
        !Array.isArray(data.features)
      ) {
        throw new Error(
          "The GIS file is not a valid GeoJSON FeatureCollection.",
        );
      }

      return data;
    },

    getStandaloneTreeSpecies(properties = {}) {
      return String(
        properties.d_CL_TREE1 ||
          properties.CL_TREE_SPECIES_NAME ||
          properties.species ||
          properties.SPECIES ||
          properties.tree_type ||
          properties.TREE_TYPE ||
          properties.QC_REMARKS ||
          properties.CL_TREE_FAMILY ||
          "",
      ).trim();
    },

    getStandaloneTreeCategory(properties = {}) {
      return String(
        properties.d_CL_TREE_ ||
          properties.CL_TREE_CATEGORY ||
          properties.category ||
          properties.CATEGORY ||
          "",
      )
        .trim()
        .toLowerCase();
    },

    getStandaloneTreeDimensions(properties = {}) {
      const category = this.getStandaloneTreeCategory(properties);
      const categoryDefaults = category.includes('"a"')
        ? { height: 12, crown: 9 }
        : category.includes('"b"')
          ? { height: 10, crown: 7 }
          : category.includes('"d"')
            ? { height: 5, crown: 3.5 }
            : { height: 8, crown: 6 };

      const heightCandidates = [
        properties.DC_MATURE_,
        properties.DC_MATURE_HEIGHT_M,
        properties.EC_TREE_HEIGHT_M,
        properties.mature_height,
        properties.height_m,
        properties.HEIGHT_M,
        properties.height,
      ];
      const crownCandidates = [
        properties.DC_MATURE1,
        properties.DC_MATURE_SPREAD_M,
        properties.EC_CANOPY_SQM
          ? Math.sqrt(Number(properties.EC_CANOPY_SQM) / Math.PI) * 2
          : null,
        properties.mature_crown,
        properties.crown_m,
        properties.CROWN_M,
        properties.canopy_m,
      ];

      const firstFinite = (values, fallback) => {
        const value = values
          .map((item) => Number(item))
          .find((item) => Number.isFinite(item) && item > 0);
        return Number.isFinite(value) ? value : fallback;
      };

      return {
        height: Math.max(
          2.5,
          Math.min(22, firstFinite(heightCandidates, categoryDefaults.height)),
        ),
        crown: Math.max(
          1.8,
          Math.min(16, firstFinite(crownCandidates, categoryDefaults.crown)),
        ),
      };
    },

    getStandaloneTreeArchetype(speciesValue) {
      const species = String(speciesValue || "").toLowerCase();

      if (
        /(acacia|vachellia|prosopis|parkinsonia|cercidium|pithecellobium|delonix|albizia)/.test(
          species,
        )
      ) {
        return "umbrella";
      }

      if (/(citrus|ziziphus|plumeria|limonia|tamarindus)/.test(species)) {
        return "round";
      }

      if (
        /(moringa|tamarix|tecomella|terminalia arjuna|dalbergia|maerua)/.test(
          species,
        )
      ) {
        return "columnar";
      }

      return "open";
    },

    getStandaloneTreePalette(speciesValue) {
      const species = String(speciesValue || "").toLowerCase();

      // Dark green canopy interiors match the concept artwork. The cyan
      // technical outline remains visible, but no longer overwhelms the fill
      // when thousands of trees are visible at once.
      if (/(tamarix|maerua|moringa)/.test(species)) {
        return {
          key: "sage",
          foliage: 0x123f31,
          emissive: 0x071f19,
        };
      }

      if (/(citrus|limonia|tamarindus|millettia)/.test(species)) {
        return {
          key: "deep",
          foliage: 0x082d20,
          emissive: 0x041a13,
        };
      }

      if (/(parkinsonia|cercidium|delonix|plumeria)/.test(species)) {
        return {
          key: "bright",
          foliage: 0x174f35,
          emissive: 0x08291d,
        };
      }

      return {
        key: "desert",
        foliage: 0x0f412c,
        emissive: 0x062219,
      };
    },

    getDeterministicTreeRotation(properties = {}, index = 0) {
      const source = String(
        properties.ID_UID ||
          properties.id ||
          properties.OBJECTID ||
          properties.FID ||
          index,
      );
      let hash = 2166136261;

      for (let position = 0; position < source.length; position += 1) {
        hash ^= source.charCodeAt(position);
        hash = Math.imul(hash, 16777619);
      }

      return ((hash >>> 0) / 4294967295) * Math.PI * 2;
    },

    createStandaloneTerrainHeightIndex(modelData) {
      if (!modelData || !modelData.pointMap) return null;
      if (modelData.terrainHeightIndex) return modelData.terrainHeightIndex;

      const cellSize = Math.max(
        5,
        Math.min(40, Number(modelData.gridSpacing) / 2 || 16),
      );
      const vertexCells = new Map();
      const triangleCells = new Map();
      const verticalExaggeration = Math.max(
        1,
        Number(modelData.verticalExaggeration) || 1,
      );
      const minimumElevation = Number(modelData.minimumElevation) || 0;

      const toHeight = (record) =>
        (Number(record.elevation) - minimumElevation) * verticalExaggeration;
      const cellKey = (cellX, cellZ) => `${cellX}|${cellZ}`;

      modelData.pointMap.forEach((record) => {
        const x = Number(record.x);
        const z = Number(record.z);
        const elevation = Number(record.elevation);
        if (![x, z, elevation].every(Number.isFinite)) return;

        const cellX = Math.floor(x / cellSize);
        const cellZ = Math.floor(z / cellSize);
        const key = cellKey(cellX, cellZ);

        if (!vertexCells.has(key)) vertexCells.set(key, []);
        vertexCells.get(key).push({ x, z, y: toHeight(record) });
      });

      // Index the actual rendered TIN triangles. Layer objects must use the
      // same triangle plane as Terrain (after), not a nearest-vertex estimate.
      // Nearest-vertex interpolation can sit several metres below a steep or
      // flattened Civil 3D face and is what allowed objects to appear under it.
      const vertexKeys = Array.isArray(modelData.triangleVertexKeys)
        ? modelData.triangleVertexKeys
        : [];

      for (let index = 0; index + 2 < vertexKeys.length; index += 3) {
        const records = [
          modelData.pointMap.get(vertexKeys[index]),
          modelData.pointMap.get(vertexKeys[index + 1]),
          modelData.pointMap.get(vertexKeys[index + 2]),
        ];
        if (records.some((record) => !record)) continue;

        const triangle = records.map((record) => ({
          x: Number(record.x),
          z: Number(record.z),
          y: toHeight(record),
        }));
        if (
          triangle.some(
            (point) => ![point.x, point.z, point.y].every(Number.isFinite),
          )
        ) {
          continue;
        }

        const minX = Math.min(...triangle.map((point) => point.x));
        const maxX = Math.max(...triangle.map((point) => point.x));
        const minZ = Math.min(...triangle.map((point) => point.z));
        const maxZ = Math.max(...triangle.map((point) => point.z));
        const minCellX = Math.floor(minX / cellSize);
        const maxCellX = Math.floor(maxX / cellSize);
        const minCellZ = Math.floor(minZ / cellSize);
        const maxCellZ = Math.floor(maxZ / cellSize);

        for (let cellX = minCellX; cellX <= maxCellX; cellX += 1) {
          for (let cellZ = minCellZ; cellZ <= maxCellZ; cellZ += 1) {
            const key = cellKey(cellX, cellZ);
            if (!triangleCells.has(key)) triangleCells.set(key, []);
            triangleCells.get(key).push(triangle);
          }
        }
      }

      modelData.terrainHeightIndex = {
        cellSize,
        vertexCells,
        triangleCells,
      };
      return modelData.terrainHeightIndex;
    },

    getStandaloneExactTerrainHeightAtPoint(point, terrainData) {
      if (!point || !terrainData) return null;

      const pointX = Number(point.x);
      const pointZ = Number(point.z);
      if (![pointX, pointZ].every(Number.isFinite)) return null;

      const index = this.createStandaloneTerrainHeightIndex(terrainData);
      if (!index) return null;

      const cellX = Math.floor(pointX / index.cellSize);
      const cellZ = Math.floor(pointZ / index.cellSize);
      const localTriangles = index.triangleCells.get(`${cellX}|${cellZ}`) || [];
      const epsilon = 1e-7;

      for (
        let triangleIndex = 0;
        triangleIndex < localTriangles.length;
        triangleIndex += 1
      ) {
        const [a, b, c] = localTriangles[triangleIndex];
        const denominator =
          (b.z - c.z) * (a.x - c.x) + (c.x - b.x) * (a.z - c.z);
        if (Math.abs(denominator) < epsilon) continue;

        const weightA =
          ((b.z - c.z) * (pointX - c.x) + (c.x - b.x) * (pointZ - c.z)) /
          denominator;
        const weightB =
          ((c.z - a.z) * (pointX - c.x) + (a.x - c.x) * (pointZ - c.z)) /
          denominator;
        const weightC = 1 - weightA - weightB;

        if (weightA >= -epsilon && weightB >= -epsilon && weightC >= -epsilon) {
          return weightA * a.y + weightB * b.y + weightC * c.y;
        }
      }

      return null;
    },

    getStandaloneNearestTerrainHeightAtPoint(point, terrainData) {
      if (!point || !terrainData) return null;
      const pointX = Number(point.x);
      const pointZ = Number(point.z);
      if (![pointX, pointZ].every(Number.isFinite)) return null;

      const index = this.createStandaloneTerrainHeightIndex(terrainData);
      if (!index) return null;

      const cellX = Math.floor(pointX / index.cellSize);
      const cellZ = Math.floor(pointZ / index.cellSize);
      const candidates = [];

      for (
        let radius = 0;
        radius <= 12 && candidates.length < 18;
        radius += 1
      ) {
        for (let xOffset = -radius; xOffset <= radius; xOffset += 1) {
          for (let zOffset = -radius; zOffset <= radius; zOffset += 1) {
            if (
              radius > 0 &&
              Math.abs(xOffset) !== radius &&
              Math.abs(zOffset) !== radius
            ) {
              continue;
            }
            const cell = index.vertexCells.get(
              `${cellX + xOffset}|${cellZ + zOffset}`,
            );
            if (cell) candidates.push(...cell);
          }
        }
      }

      if (!candidates.length) return null;

      const nearest = candidates
        .map((sample) => {
          const dx = sample.x - pointX;
          const dz = sample.z - pointZ;
          return { sample, distanceSquared: dx * dx + dz * dz };
        })
        .sort((a, b) => a.distanceSquared - b.distanceSquared)
        .slice(0, 6);

      if (!nearest.length) return null;
      if (nearest[0].distanceSquared < 0.0001) {
        return nearest[0].sample.y;
      }

      let weightedHeight = 0;
      let totalWeight = 0;
      nearest.forEach((item) => {
        const weight = 1 / Math.max(0.25, item.distanceSquared);
        weightedHeight += item.sample.y * weight;
        totalWeight += weight;
      });
      return totalWeight > 0 ? weightedHeight / totalWeight : null;
    },

    convertStandaloneTerrainHeightBetweenModels(
      localHeight,
      sourceData,
      targetData,
    ) {
      if (!Number.isFinite(Number(localHeight)) || !sourceData || !targetData) {
        return null;
      }
      const sourceExaggeration = Math.max(
        0.0001,
        Number(sourceData.verticalExaggeration) || 1,
      );
      const targetExaggeration = Math.max(
        0.0001,
        Number(targetData.verticalExaggeration) || 1,
      );
      const absoluteElevation =
        Number(localHeight) / sourceExaggeration +
        (Number(sourceData.minimumElevation) || 0);
      return (
        (absoluteElevation - (Number(targetData.minimumElevation) || 0)) *
        targetExaggeration
      );
    },

    getStandaloneTerrainHeightAtPoint(point, modelData = null) {
      // All ArcGIS objects are retained, including objects beyond the visible
      // floating-island boundary. Prefer the full Civil 3D Terrain (after) TIN.
      // Where the LandXML genuinely has no face, use Terrain (before) as a
      // geographic support surface, translated into Terrain (after)'s vertical
      // coordinate system. Only then use the nearest Civil 3D edge as a final
      // visual fallback so an object can never fall to y=0 or sink below ground.
      const rootData = modelData || this.threeParkModelData;
      if (!point || !rootData) return 0;

      const afterData = rootData.terrainAfterData || rootData;
      const beforeData =
        rootData.terrainBeforeData ||
        (this.threeParkModelData &&
          this.threeParkModelData.terrainBeforeData) ||
        null;

      const exactAfter = this.getStandaloneExactTerrainHeightAtPoint(
        point,
        afterData,
      );
      if (Number.isFinite(exactAfter)) return exactAfter;

      if (beforeData && beforeData !== afterData) {
        const exactBefore = this.getStandaloneExactTerrainHeightAtPoint(
          point,
          beforeData,
        );
        if (Number.isFinite(exactBefore)) {
          const converted = this.convertStandaloneTerrainHeightBetweenModels(
            exactBefore,
            beforeData,
            afterData,
          );
          if (Number.isFinite(converted)) return converted;
        }
      }

      // Try Mapbox's terrain source directly for ArcGIS objects beyond both
      // triangulated meshes. This is useful where the GIS export extends beyond
      // the Civil 3D TIN boundary.
      const origin = rootData.origin || afterData.origin;
      if (origin) {
        const lngLat = this.terrainMetersToLngLat(point, origin);
        const mapboxElevation = this.queryStandaloneTerrainElevation(lngLat);
        if (Number.isFinite(Number(mapboxElevation))) {
          return (
            (Number(mapboxElevation) -
              (Number(afterData.minimumElevation) || 0)) *
            Math.max(0.0001, Number(afterData.verticalExaggeration) || 1)
          );
        }
      }

      const nearestAfter = this.getStandaloneNearestTerrainHeightAtPoint(
        point,
        afterData,
      );
      if (Number.isFinite(nearestAfter)) return nearestAfter;

      if (beforeData && beforeData !== afterData) {
        const nearestBefore = this.getStandaloneNearestTerrainHeightAtPoint(
          point,
          beforeData,
        );
        if (Number.isFinite(nearestBefore)) {
          const converted = this.convertStandaloneTerrainHeightBetweenModels(
            nearestBefore,
            beforeData,
            afterData,
          );
          if (Number.isFinite(converted)) return converted;
        }
      }

      return 0;
    },

    getStandaloneTerrainSupportHeightAtPoint(
      point,
      modelData = null,
      footprintRadius = 0,
    ) {
      const centerHeight = this.getStandaloneTerrainHeightAtPoint(
        point,
        modelData,
      );
      const radius = Math.max(0, Number(footprintRadius) || 0);
      if (!radius) return centerHeight;

      // Point assets can straddle a steep Terrain (after) TIN face or sit
      // directly on the clipped park boundary. Use the highest terrain sample
      // beneath the small model footprint so the visible trunk/pole base is
      // never swallowed by an adjacent triangle.
      const offsets = [
        [radius, 0],
        [-radius, 0],
        [0, radius],
        [0, -radius],
        [radius * 0.7071, radius * 0.7071],
        [-radius * 0.7071, radius * 0.7071],
        [radius * 0.7071, -radius * 0.7071],
        [-radius * 0.7071, -radius * 0.7071],
      ];
      let supportHeight = centerHeight;
      offsets.forEach(([offsetX, offsetZ]) => {
        const sampledHeight = this.getStandaloneTerrainHeightAtPoint(
          {
            x: Number(point.x) + offsetX,
            z: Number(point.z) + offsetZ,
          },
          modelData,
        );
        if (Number.isFinite(sampledHeight)) {
          supportHeight = Math.max(supportHeight, sampledHeight);
        }
      });
      return supportHeight;
    },

    updateStandaloneVegetationLineLod() {
      if (!this.threeCamera || !this.threeControls || !this.threeLayerGroups) {
        return;
      }
      const vegetation = this.threeLayerGroups.vegetation;
      if (!vegetation || !vegetation.visible) return;

      const target = this.threeControls.target || new THREE.Vector3();
      const distance = this.threeCamera.position.distanceTo(target);
      const maxDimension = Math.max(
        1,
        Number(
          this.threeParkModelData && this.threeParkModelData.maxDimension,
        ) || 500,
      );
      // 0 = close view, 1 = far overview. Fade the wireframe as the map gets
      // smaller so thousands of cyan/white edges cannot merge into a white mass.
      const farRatio = THREE.MathUtils.clamp(
        (distance - maxDimension * 0.55) / (maxDimension * 1.9),
        0,
        1,
      );

      vegetation.traverse((object) => {
        if (!object || !object.material) return;
        const name = String(object.name || "");
        const materials = Array.isArray(object.material)
          ? object.material
          : [object.material];

        materials.forEach((material) => {
          if (!material || !material.transparent) return;
          if (
            name.includes("-outline") ||
            name === "vegetation-trunk-outlines"
          ) {
            material.opacity = THREE.MathUtils.lerp(0.38, 0.045, farRatio);
            material.needsUpdate = true;
          } else if (name.includes("-glow")) {
            material.opacity = THREE.MathUtils.lerp(0.065, 0.008, farRatio);
            material.needsUpdate = true;
          }
        });
      });
    },

    createStandaloneTreeCanopyGeometry(archetype) {
      if (archetype === "umbrella") {
        return new THREE.SphereGeometry(1, 9, 6);
      }

      if (archetype === "round") {
        return new THREE.IcosahedronGeometry(1, 1);
      }

      if (archetype === "columnar") {
        return new THREE.ConeGeometry(1, 2, 9, 2, false);
      }

      return new THREE.DodecahedronGeometry(1, 0);
    },

    async createStandaloneVegetationGroup(geojson, modelData, loadToken) {
      const group = new THREE.Group();
      group.name = "standalone-vegetation";
      const origin = modelData && modelData.origin;
      const localParts = modelData && modelData.localParts;
      const features =
        geojson && Array.isArray(geojson.features) ? geojson.features : [];
      const treeRecords = [];
      const nonTreeFeatures = [];

      features.forEach((feature, index) => {
        const geometry = feature && feature.geometry;
        if (!geometry) return;

        if (geometry.type !== "Point" && geometry.type !== "MultiPoint") {
          nonTreeFeatures.push(feature);
          return;
        }

        const coordinates =
          geometry.type === "Point"
            ? [geometry.coordinates]
            : geometry.coordinates;

        coordinates.forEach((coordinate, coordinateIndex) => {
          if (
            !Array.isArray(coordinate) ||
            coordinate.length < 2 ||
            !coordinate.slice(0, 2).map(Number).every(Number.isFinite)
          ) {
            return;
          }

          const local = this.lngLatToTerrainMeters(coordinate, origin);
          const properties = feature.properties || {};
          const species = this.getStandaloneTreeSpecies(properties);
          const dimensions = this.getStandaloneTreeDimensions(properties);
          const palette = this.getStandaloneTreePalette(species);

          treeRecords.push({
            index: index * 10 + coordinateIndex,
            local,
            species,
            archetype: this.getStandaloneTreeArchetype(species),
            palette,
            height: dimensions.height,
            crown: dimensions.crown,
            rotation: this.getDeterministicTreeRotation(
              properties,
              index * 10 + coordinateIndex,
            ),
          });
        });
      });

      if (!treeRecords.length && !nonTreeFeatures.length) {
        throw new Error(
          "The vegetation GeoJSON contains no usable vegetation features.",
        );
      }

      let processedBuckets = 0;

      if (treeRecords.length) {
        const trunkGeometry = new THREE.CylinderGeometry(0.5, 0.62, 1, 7, 1);
        // Use an unlit holographic material so the trunk keeps the same
        // green/cyan colour from bird's-eye and underside camera angles.
        const trunkMaterial = new THREE.MeshBasicMaterial({
          color: 0x052f26,
          transparent: true,
          opacity: 0.82,
          side: THREE.DoubleSide,
          depthWrite: true,
          depthTest: true,
        });
        const trunks = new THREE.InstancedMesh(
          trunkGeometry,
          trunkMaterial,
          treeRecords.length,
        );
        trunks.name = "vegetation-trunks";
        trunks.castShadow = false;
        trunks.receiveShadow = true;
        trunks.instanceMatrix.setUsage(THREE.DynamicDrawUsage);

        const trunkOutlineMaterial = new THREE.MeshBasicMaterial({
          color: 0x54cbd0,
          wireframe: true,
          transparent: true,
          opacity: 0.18,
          depthWrite: false,
          depthTest: true,
          blending: THREE.NormalBlending,
        });
        const trunkOutlines = new THREE.InstancedMesh(
          trunkGeometry,
          trunkOutlineMaterial,
          treeRecords.length,
        );
        trunkOutlines.name = "vegetation-trunk-outlines";
        trunkOutlines.instanceMatrix.setUsage(THREE.DynamicDrawUsage);
        trunkOutlines.renderOrder = 10;

        const buckets = new Map();
        treeRecords.forEach((tree) => {
          const key = `${tree.archetype}|${tree.palette.key}`;
          if (!buckets.has(key)) {
            buckets.set(key, {
              archetype: tree.archetype,
              palette: tree.palette,
              trees: [],
            });
          }
          buckets.get(key).trees.push(tree);
        });

        const dummy = new THREE.Object3D();

        treeRecords.forEach((tree, index) => {
          const baseY =
            this.getStandaloneTerrainSupportHeightAtPoint(
              tree.local,
              modelData,
              Math.max(0.28, Math.min(0.65, tree.crown * 0.045)),
            ) + 0.16;
          const trunkHeight = Math.max(
            1.2,
            tree.height *
              (tree.archetype === "umbrella"
                ? 0.46
                : tree.archetype === "columnar"
                  ? 0.52
                  : 0.42),
          );
          const trunkRadius = Math.max(
            0.12,
            Math.min(0.55, tree.crown * 0.035),
          );

          tree.baseY = baseY;
          tree.trunkHeight = trunkHeight;
          tree.canopyHeight = Math.max(1.2, tree.height - trunkHeight);

          dummy.position.set(
            tree.local.x,
            baseY + trunkHeight / 2,
            tree.local.z,
          );
          dummy.rotation.set(0, tree.rotation, 0);
          dummy.scale.set(trunkRadius * 2, trunkHeight, trunkRadius * 2);
          dummy.updateMatrix();
          trunks.setMatrixAt(index, dummy.matrix);

          // A very small scale increase avoids z-fighting and gives the trunk the
          // bright cyan wire silhouette visible in the concept artwork.
          dummy.scale.set(
            trunkRadius * 2.08,
            trunkHeight * 1.01,
            trunkRadius * 2.08,
          );
          dummy.updateMatrix();
          trunkOutlines.setMatrixAt(index, dummy.matrix);
        });
        trunks.instanceMatrix.needsUpdate = true;
        trunkOutlines.instanceMatrix.needsUpdate = true;
        group.add(trunks);
        group.add(trunkOutlines);

        buckets.forEach((bucket) => {
          const geometry = this.createStandaloneTreeCanopyGeometry(
            bucket.archetype,
          );
          // MeshBasicMaterial is intentional here. The previous physically-lit
          // material was overexposed by the scene's key/hemisphere lights, so
          // canopies appeared white from above while their colour was visible
          // mainly from below. This preserves the GIS palette on both sides.
          const material = new THREE.MeshBasicMaterial({
            color: bucket.palette.foliage,
            transparent: true,
            opacity: 0.8,
            side: THREE.DoubleSide,
            depthWrite: true,
            depthTest: true,
          });
          const canopy = new THREE.InstancedMesh(
            geometry,
            material,
            bucket.trees.length,
          );
          canopy.name = `vegetation-${bucket.archetype}-${bucket.palette.key}`;
          canopy.instanceMatrix.setUsage(THREE.DynamicDrawUsage);

          const outlineMaterial = new THREE.MeshBasicMaterial({
            color: 0x54cbd0,
            wireframe: true,
            transparent: true,
            opacity: 0.22,
            depthWrite: false,
            depthTest: true,
            blending: THREE.NormalBlending,
          });
          const outline = new THREE.InstancedMesh(
            geometry,
            outlineMaterial,
            bucket.trees.length,
          );
          outline.name = `${canopy.name}-outline`;
          outline.instanceMatrix.setUsage(THREE.DynamicDrawUsage);
          outline.renderOrder = 9;

          const glowMaterial = new THREE.MeshBasicMaterial({
            color: 0x1aaab5,
            wireframe: true,
            transparent: true,
            opacity: 0.03,
            depthWrite: false,
            depthTest: true,
            blending: THREE.AdditiveBlending,
          });
          const glow = new THREE.InstancedMesh(
            geometry,
            glowMaterial,
            bucket.trees.length,
          );
          glow.name = `${canopy.name}-glow`;
          glow.instanceMatrix.setUsage(THREE.DynamicDrawUsage);
          glow.renderOrder = 8;

          bucket.trees.forEach((tree, index) => {
            const crownRadius = Math.max(0.8, tree.crown / 2);
            let scaleX = crownRadius;
            let scaleY = Math.max(0.8, tree.canopyHeight / 2);
            let scaleZ = crownRadius;
            let centerY =
              tree.baseY + tree.trunkHeight + tree.canopyHeight * 0.42;

            if (tree.archetype === "umbrella") {
              scaleY = Math.max(0.65, tree.canopyHeight * 0.27);
              centerY =
                tree.baseY + tree.trunkHeight + tree.canopyHeight * 0.24;
            } else if (tree.archetype === "columnar") {
              scaleX = crownRadius * 0.62;
              scaleZ = crownRadius * 0.62;
              scaleY = Math.max(1, tree.canopyHeight / 2);
              centerY = tree.baseY + tree.trunkHeight + tree.canopyHeight * 0.5;
            } else if (tree.archetype === "open") {
              scaleY = Math.max(0.75, tree.canopyHeight * 0.36);
            }

            dummy.position.set(tree.local.x, centerY, tree.local.z);
            dummy.rotation.set(0, tree.rotation, 0);
            dummy.scale.set(scaleX, scaleY, scaleZ);
            dummy.updateMatrix();
            canopy.setMatrixAt(index, dummy.matrix);

            dummy.scale.set(scaleX * 1.025, scaleY * 1.025, scaleZ * 1.025);
            dummy.updateMatrix();
            outline.setMatrixAt(index, dummy.matrix);

            dummy.scale.set(scaleX * 1.085, scaleY * 1.085, scaleZ * 1.085);
            dummy.updateMatrix();
            glow.setMatrixAt(index, dummy.matrix);
          });

          canopy.instanceMatrix.needsUpdate = true;
          outline.instanceMatrix.needsUpdate = true;
          glow.instanceMatrix.needsUpdate = true;
          group.add(canopy);
          group.add(glow);
          group.add(outline);
          processedBuckets += 1;
        });
      }

      if (nonTreeFeatures.length) {
        const nonTreeGeoJson = {
          type: "FeatureCollection",
          features: nonTreeFeatures,
        };
        const plantingGroup = await this.createStandaloneGenericFeatureGroup(
          "vegetation",
          nonTreeGeoJson,
          modelData,
        );
        plantingGroup.name = "vegetation-shrubs-groundcover";
        group.userData.shrubCount = plantingGroup.userData.shrubCount || 0;
        group.userData.groundCoverCount = nonTreeFeatures.filter((feature) =>
          /ground[_ -]?cover|lawn|turf|grass/.test(
            this.getStandaloneVegetationDescriptor(
              this.getStandaloneFeatureType(feature),
              (feature && feature.properties) || {},
            ),
          ),
        ).length;
        group.add(plantingGroup);
      }

      group.userData.treeCount = treeRecords.length;
      group.userData.speciesCount = new Set(
        treeRecords.map((tree) => tree.species || "Unknown"),
      ).size;
      group.userData.bucketCount = processedBuckets;

      if (loadToken !== this.threeVegetationLoadToken) {
        this.disposeStandaloneThreeGroup(group);
        throw new Error("The vegetation load was cancelled.");
      }

      return group;
    },

    getStandaloneFeatureType(feature) {
      const properties = (feature && feature.properties) || {};
      return String(
        properties.featureType ||
          properties.feature_type ||
          properties.subtype ||
          properties.type ||
          properties.QC_REMARKS ||
          properties.Remarks ||
          "",
      )
        .trim()
        .toLowerCase();
    },

    getStandaloneNumericProperty(properties = {}, keys = [], fallback = 0) {
      for (const key of keys) {
        const value = Number(properties[key]);
        if (Number.isFinite(value) && value > 0) return value;
      }
      return fallback;
    },

    getStandaloneVegetationDescriptor(featureType = "", properties = {}) {
      return [
        featureType,
        properties.featureType,
        properties.sourceLayer,
        properties.source_layer,
        properties._source_layer,
        properties.QC_REMARKS,
        properties.DC_REMARK,
        properties.Remarks,
        properties.species,
        properties.SPECIES,
        properties.CL_SHRUB_FAMILY,
        properties.CL_SHRUB_GENUS,
      ]
        .filter((value) => value != null && String(value).trim())
        .join(" ")
        .toLowerCase()
        .replace(/[_-]+/g, " ");
    },

    isStandaloneShrubFeature(featureType = "", properties = {}) {
      return /\b(shrub|shrubs|bush|bushes)\b/.test(
        this.getStandaloneVegetationDescriptor(featureType, properties),
      );
    },

    getStandaloneShrubDimensions(properties = {}, localRings = []) {
      const outer = Array.isArray(localRings) ? localRings[0] : null;
      const bounds = this.getStandalonePolygonBounds(outer || []);
      const footprintSpread = bounds
        ? Math.max(bounds.maxX - bounds.minX, bounds.maxY - bounds.minY)
        : 0;
      const explicitSpread = this.getStandaloneNumericProperty(
        properties,
        [
          "DC_MATURE_SPREAD_M",
          "DC_MATURE_SPREAD",
          "mature_spread_m",
          "spread_m",
          "SPREAD_M",
          "canopy_m",
        ],
        0,
      );
      const spread = Math.max(
        0.65,
        Math.min(8, explicitSpread || footprintSpread || 1.8),
      );
      const explicitHeight = this.getStandaloneNumericProperty(
        properties,
        [
          "DC_MATURE_HEIGHT_M",
          "DC_MATURE_HEIGHT",
          "mature_height_m",
          "height_m",
          "HEIGHT_M",
          "height",
        ],
        0,
      );

      const descriptor = this.getStandaloneVegetationDescriptor(
        "shrub",
        properties,
      );
      const isClimber =
        /boug(?:ain|an)villea|jasminum|quisqualis|quiscualis|senecio/.test(
          descriptor,
        );
      const requestedHeight = explicitHeight || Math.max(1.2, spread * 0.72);

      return {
        // DC_MATURE_HEIGHT_M is used directly for ordinary shrubs. For climbing
        // plants, the GIS value can describe total vine length (often 10 m), not
        // the unsupported vertical height of a free-standing bush. Keep those
        // plants shrub-sized until a pergola/trellis support is modelled.
        height: Math.max(
          0.85,
          Math.min(
            isClimber ? Math.max(3.2, spread * 0.72) : 5.5,
            requestedHeight,
          ),
        ),
        // The planned mature spread is the visible crown diameter. A small
        // minimum keeps the 1 m shrubs readable at park-wide zoom without
        // changing the relative 1 m / 2 m / 3 m GIS sizing.
        spread: Math.max(1.15, spread),
      };
    },

    getStandaloneShrubArchetype(properties = {}) {
      const species = this.getStandaloneVegetationDescriptor(
        "shrub",
        properties,
      );

      if (/crinum/.test(species)) return "spiky";
      if (/caligonum|haloxylon/.test(species)) return "airy";
      if (
        /boug(?:ain|an)villea|jasminum|quisqualis|quiscualis|senecio/.test(
          species,
        )
      ) {
        return "climbing";
      }
      if (/clerodendrum|dodonea|vitex/.test(species)) return "upright";
      return "mounded";
    },

    getStandaloneShrubFlowerColour(properties = {}) {
      const species = this.getStandaloneVegetationDescriptor(
        "shrub",
        properties,
      );

      if (/boug(?:ain|an)villea/.test(species)) return 0xd64b91;
      if (/senecio/.test(species)) return 0xe27632;
      if (/quisqualis|quiscualis/.test(species)) return 0xeeb4bd;
      if (/jasminum|crinum/.test(species)) return 0xf1eee2;
      return null;
    },

    createStandaloneShrubInstanceObjects(records, modelData, materials) {
      const group = new THREE.Group();
      group.name = "vegetation-realistic-shrubs";
      const roundLobes = [];
      const spikyLeaves = [];
      const flowersByColour = new Map();
      let shrubCount = 0;

      const lobeTemplates = {
        mounded: [
          { x: 0, z: 0, centerY: 0.46, sx: 0.48, sy: 0.46, sz: 0.46 },
          { x: -0.23, z: 0.09, centerY: 0.32, sx: 0.34, sy: 0.31, sz: 0.33 },
          { x: 0.23, z: -0.08, centerY: 0.34, sx: 0.35, sy: 0.33, sz: 0.34 },
          { x: 0.04, z: 0.23, centerY: 0.27, sx: 0.32, sy: 0.26, sz: 0.31 },
          { x: -0.05, z: -0.23, centerY: 0.26, sx: 0.31, sy: 0.25, sz: 0.3 },
        ],
        airy: [
          { x: 0, z: 0, centerY: 0.48, sx: 0.27, sy: 0.47, sz: 0.25 },
          { x: -0.25, z: 0.08, centerY: 0.33, sx: 0.22, sy: 0.31, sz: 0.2 },
          { x: 0.25, z: -0.08, centerY: 0.36, sx: 0.23, sy: 0.34, sz: 0.21 },
          { x: -0.08, z: -0.24, centerY: 0.27, sx: 0.2, sy: 0.25, sz: 0.2 },
          { x: 0.1, z: 0.24, centerY: 0.3, sx: 0.21, sy: 0.28, sz: 0.2 },
        ],
        upright: [
          { x: 0, z: 0, centerY: 0.5, sx: 0.27, sy: 0.5, sz: 0.26 },
          { x: -0.18, z: 0.05, centerY: 0.43, sx: 0.24, sy: 0.4, sz: 0.23 },
          { x: 0.18, z: -0.05, centerY: 0.45, sx: 0.24, sy: 0.42, sz: 0.23 },
          { x: 0.03, z: 0.18, centerY: 0.35, sx: 0.23, sy: 0.33, sz: 0.22 },
        ],
        climbing: [
          { x: 0, z: 0, centerY: 0.5, sx: 0.26, sy: 0.5, sz: 0.25 },
          { x: -0.17, z: 0.05, centerY: 0.32, sx: 0.23, sy: 0.3, sz: 0.22 },
          { x: 0.16, z: -0.05, centerY: 0.51, sx: 0.22, sy: 0.37, sz: 0.21 },
          { x: 0.02, z: 0.13, centerY: 0.73, sx: 0.2, sy: 0.25, sz: 0.2 },
        ],
      };

      (Array.isArray(records) ? records : []).forEach(
        ({ feature, featureType }, recordIndex) => {
          const geometry = feature && feature.geometry;
          const properties = (feature && feature.properties) || {};
          const archetype = this.getStandaloneShrubArchetype(properties);
          const flowerColour = this.getStandaloneShrubFlowerColour(properties);

          this.getStandalonePolygonParts(geometry).forEach(
            (rings, partIndex) => {
              const localRings = this.getStandaloneLocalPolygonRings(
                rings,
                modelData.origin,
              );
              if (!localRings.length) return;

              const outer = localRings[0];
              let center = this.getStandalonePolygonCentroid(outer);
              if (!center) return;
              if (!this.isStandalonePointInsidePolygonRing(center, outer)) {
                center = outer[0].clone();
              }

              const dimensions = this.getStandaloneShrubDimensions(
                properties,
                localRings,
              );
              const baseY =
                this.getStandaloneTerrainSupportHeightAtPoint(
                  { x: center.x, z: center.y },
                  modelData,
                  Math.min(1.8, dimensions.spread * 0.32),
                ) + 0.045;
              const rotation = this.getDeterministicTreeRotation(
                properties,
                recordIndex * 7 + partIndex,
              );
              const cosine = Math.cos(rotation);
              const sine = Math.sin(rotation);

              if (archetype === "spiky") {
                const leafCount = 9;
                for (let leafIndex = 0; leafIndex < leafCount; leafIndex += 1) {
                  const angle =
                    rotation + (leafIndex / leafCount) * Math.PI * 2;
                  const lean = 0.22 + (leafIndex % 3) * 0.055;
                  spikyLeaves.push({
                    x: center.x + Math.cos(angle) * dimensions.spread * 0.11,
                    y: baseY + dimensions.height * 0.42,
                    z: center.y + Math.sin(angle) * dimensions.spread * 0.11,
                    rotationY: angle,
                    rotationZ: Math.cos(angle) * lean,
                    rotationX: Math.sin(angle) * lean,
                    scaleX: dimensions.spread * 0.075,
                    scaleY: dimensions.height * (0.38 + (leafIndex % 2) * 0.06),
                    scaleZ: dimensions.spread * 0.045,
                    shade: leafIndex % 3,
                  });
                }
              } else {
                const templates =
                  lobeTemplates[archetype] || lobeTemplates.mounded;
                templates.forEach((template, templateIndex) => {
                  const offsetX = template.x * dimensions.spread;
                  const offsetZ = template.z * dimensions.spread;
                  roundLobes.push({
                    x: center.x + offsetX * cosine - offsetZ * sine,
                    y: baseY + template.centerY * dimensions.height,
                    z: center.y + offsetX * sine + offsetZ * cosine,
                    rotationY: rotation + templateIndex * 0.37,
                    scaleX: Math.max(
                      0.28,
                      template.sx * dimensions.spread * 1.08,
                    ),
                    scaleY: Math.max(0.3, template.sy * dimensions.height),
                    scaleZ: Math.max(
                      0.28,
                      template.sz * dimensions.spread * 1.08,
                    ),
                    shade: (recordIndex + templateIndex) % 3,
                  });
                });
              }

              if (flowerColour != null) {
                const flowerKey = String(flowerColour);
                if (!flowersByColour.has(flowerKey)) {
                  flowersByColour.set(flowerKey, {
                    colour: flowerColour,
                    records: [],
                  });
                }
                const flowers = flowersByColour.get(flowerKey).records;
                const flowerCount = dimensions.height > 4 ? 5 : 3;
                for (
                  let flowerIndex = 0;
                  flowerIndex < flowerCount;
                  flowerIndex += 1
                ) {
                  const angle =
                    rotation +
                    (flowerIndex / flowerCount) * Math.PI * 2 +
                    (recordIndex % 5) * 0.19;
                  flowers.push({
                    x:
                      center.x +
                      Math.cos(angle) *
                        dimensions.spread *
                        (0.17 + 0.025 * flowerIndex),
                    y:
                      baseY +
                      dimensions.height * (0.58 + 0.075 * (flowerIndex % 3)),
                    z:
                      center.y +
                      Math.sin(angle) *
                        dimensions.spread *
                        (0.17 + 0.025 * flowerIndex),
                    scale: Math.max(
                      0.055,
                      Math.min(0.24, dimensions.spread * 0.055),
                    ),
                  });
                }
              }

              shrubCount += 1;
            },
          );
        },
      );

      const sourceColour =
        materials && materials.solid && materials.solid.color
          ? materials.solid.color.clone()
          : new THREE.Color(0x496947);
      const dummy = new THREE.Object3D();

      if (roundLobes.length) {
        const geometry = new THREE.IcosahedronGeometry(1, 2);
        const material = new THREE.MeshStandardMaterial({
          color: 0xffffff,
          emissive: sourceColour.clone().multiplyScalar(0.06),
          emissiveIntensity: 0.08,
          roughness: 1,
          metalness: 0,
          flatShading: false,
        });
        const mesh = new THREE.InstancedMesh(
          geometry,
          material,
          roundLobes.length,
        );
        mesh.name = "vegetation-shrub-foliage";
        mesh.frustumCulled = false;
        mesh.renderOrder = 15;

        roundLobes.forEach((lobe, index) => {
          dummy.position.set(lobe.x, lobe.y, lobe.z);
          dummy.rotation.set(0, lobe.rotationY, 0);
          dummy.scale.set(lobe.scaleX, lobe.scaleY, lobe.scaleZ);
          dummy.updateMatrix();
          mesh.setMatrixAt(index, dummy.matrix);

          const colour = sourceColour.clone();
          colour.offsetHSL(
            lobe.shade === 1 ? -0.008 : lobe.shade === 2 ? 0.008 : 0,
            lobe.shade === 1 ? -0.035 : 0.015,
            lobe.shade === 1 ? -0.045 : lobe.shade === 2 ? 0.035 : 0,
          );
          mesh.setColorAt(index, colour);
        });

        mesh.instanceMatrix.needsUpdate = true;
        if (mesh.instanceColor) mesh.instanceColor.needsUpdate = true;
        group.add(mesh);
      }

      if (spikyLeaves.length) {
        const geometry = new THREE.ConeGeometry(1, 2, 6, 1, false);
        const material = new THREE.MeshStandardMaterial({
          color: 0xffffff,
          emissive: sourceColour.clone().multiplyScalar(0.14),
          emissiveIntensity: 0.18,
          roughness: 0.96,
          metalness: 0,
          flatShading: false,
        });
        const mesh = new THREE.InstancedMesh(
          geometry,
          material,
          spikyLeaves.length,
        );
        mesh.name = "vegetation-shrub-spiky-leaves";
        mesh.frustumCulled = false;
        mesh.renderOrder = 15;

        spikyLeaves.forEach((leaf, index) => {
          dummy.position.set(leaf.x, leaf.y, leaf.z);
          dummy.rotation.set(leaf.rotationX, leaf.rotationY, leaf.rotationZ);
          dummy.scale.set(leaf.scaleX, leaf.scaleY, leaf.scaleZ);
          dummy.updateMatrix();
          mesh.setMatrixAt(index, dummy.matrix);

          const colour = sourceColour.clone();
          colour.offsetHSL(
            leaf.shade === 1 ? -0.01 : leaf.shade === 2 ? 0.01 : 0,
            -0.02,
            leaf.shade === 1 ? -0.04 : leaf.shade === 2 ? 0.035 : 0,
          );
          mesh.setColorAt(index, colour);
        });

        mesh.instanceMatrix.needsUpdate = true;
        if (mesh.instanceColor) mesh.instanceColor.needsUpdate = true;
        group.add(mesh);
      }

      flowersByColour.forEach(({ colour, records: flowerRecords }) => {
        if (!flowerRecords.length) return;
        const geometry = new THREE.IcosahedronGeometry(1, 1);
        const material = new THREE.MeshStandardMaterial({
          color: colour,
          emissive: colour,
          emissiveIntensity: 0.12,
          roughness: 0.82,
          metalness: 0,
        });
        const mesh = new THREE.InstancedMesh(
          geometry,
          material,
          flowerRecords.length,
        );
        mesh.name = `vegetation-shrub-flowers-${colour}`;
        mesh.frustumCulled = false;
        mesh.renderOrder = 16;

        flowerRecords.forEach((flower, index) => {
          dummy.position.set(flower.x, flower.y, flower.z);
          dummy.rotation.set(0, index * 0.63, 0);
          dummy.scale.setScalar(flower.scale);
          dummy.updateMatrix();
          mesh.setMatrixAt(index, dummy.matrix);
        });
        mesh.instanceMatrix.needsUpdate = true;
        group.add(mesh);
      });

      group.userData.shrubCount = shrubCount;
      return group;
    },

    getStandaloneLocalPolygonRings(rings, origin) {
      if (!Array.isArray(rings)) return [];

      return rings
        .map((ring) => {
          const points = (Array.isArray(ring) ? ring : [])
            .filter(
              (coordinate) =>
                Array.isArray(coordinate) &&
                Number.isFinite(Number(coordinate[0])) &&
                Number.isFinite(Number(coordinate[1])),
            )
            .map((coordinate) => {
              const point = this.lngLatToTerrainMeters(coordinate, origin);
              return new THREE.Vector2(point.x, point.z);
            });

          if (points.length > 1) {
            const first = points[0];
            const last = points[points.length - 1];
            if (first.distanceToSquared(last) < 0.000001) points.pop();
          }

          return points;
        })
        .filter((ring) => ring.length >= 3);
    },

    getStandalonePolygonParts(geometry) {
      if (!geometry || !Array.isArray(geometry.coordinates)) return [];
      if (geometry.type === "Polygon") return [geometry.coordinates];
      if (geometry.type === "MultiPolygon") return geometry.coordinates;
      return [];
    },

    getStandaloneLineStrings(geometry) {
      if (!geometry || !Array.isArray(geometry.coordinates)) return [];
      if (geometry.type === "LineString") return [geometry.coordinates];
      if (geometry.type === "MultiLineString") return geometry.coordinates;
      return [];
    },

    getStandaloneStructureHeight(featureType, properties = {}) {
      const explicit = this.getStandaloneNumericProperty(
        properties,
        [
          "height_m",
          "Height_M",
          "HEIGHT_M",
          "Height",
          "HEIGHT",
          "height",
          "PoleHeight",
          "DC_HEIGHT",
        ],
        0,
      );
      const architecturalBuildingStyle =
        this.getStandaloneArchitecturalBuildingStyle(featureType, properties);
      if (architecturalBuildingStyle) {
        const architecturalHeight =
          Number(architecturalBuildingStyle.wallHeight || 0) +
          Number(architecturalBuildingStyle.roofThickness || 0) +
          Number(architecturalBuildingStyle.parapetHeight || 0);
        return Math.max(
          Math.max(0, Math.min(30, explicit || 0)),
          Math.max(0.12, architecturalHeight),
        );
      }

      const detailedModularStyle =
        this.getStandaloneDetailedModularBuildingStyle(featureType, properties);
      if (detailedModularStyle) {
        const designHeight =
          Number(detailedModularStyle.eaveHeight || 0) +
          Number(detailedModularStyle.ridgeRise || 0) +
          Number(detailedModularStyle.crossRise || 0);
        return Math.max(
          Math.max(0, Math.min(30, explicit || 0)),
          Math.max(0.12, designHeight),
        );
      }

      const detailedEquipmentStyle = this.getStandaloneDetailedEquipmentStyle(
        featureType,
        properties,
      );
      if (detailedEquipmentStyle) {
        if (detailedEquipmentStyle.kind === "running-track") return 0.18;
        if (detailedEquipmentStyle.kind === "petanque") return 0.4;
        if (detailedEquipmentStyle.kind === "bicycle-rack") return 0.95;
        if (detailedEquipmentStyle.kind === "table-tennis") return 0.92;
        if (detailedEquipmentStyle.kind === "workout") return 2.25;
      }

      if (explicit > 0) return Math.max(0.12, Math.min(30, explicit));

      // The court slab is low, but the complete structure includes a
      // regulation-height hoop and backboard. This height is also used by
      // labels and pergola-clearance checks.
      if (this.isStandaloneHalfBasketballCourtStructure(properties)) {
        return 4.15;
      }

      // The skatepark is a recessed concrete bowl contained by a raised
      // perimeter slab. The procedural model itself sets the precise height,
      // while this value keeps labels and clearance checks above the rim.
      if (this.isStandaloneSkateparkStructure(properties)) {
        return 2.2;
      }

      const type = String(featureType || "").toLowerCase();
      if (/sport|court|parking|pedestrian|median/.test(type)) return 0.18;
      if (/bench/.test(type)) return 0.72;
      if (/bin|recycling/.test(type)) return 1.0;
      if (/bike|rack/.test(type)) return 1.15;
      if (/table_tenis|table_tennis/.test(type)) return 0.78;
      if (/playground|outdoor_gym|petanque/.test(type)) return 1.9;
      if (/pergola/.test(type)) return 3.2;
      if (/kiosk|toilet|prayer|electric_room/.test(type)) return 3.8;
      if (/visitor|restaurant|administration|maintenance/.test(type)) {
        return 5.5;
      }
      return 1.25;
    },

    getStandaloneLayerPalette(layerKey) {
      const palettes = {
        paths: {
          // Fallback colour for PATHS features that are not one of the three
          // concrete classes below (for example cycle paths or landcover).
          solid: 0x07566b,
          emissive: 0x022d39,
          line: 0x67cbd2,
          opacity: 1,
          wireOpacity: 0.18,
          lineOpacity: 0.36,
        },
        water: {
          // Deep blue water; internal wire/grid lines are intentionally subtle.
          solid: 0x032f58,
          emissive: 0x021d38,
          line: 0x75cfe8,
          opacity: 1,
          wireOpacity: 0.035,
          lineOpacity: 0.08,
        },
        structures: {
          // Keep structures purple, but dim their technical outlines so dense
          // groups do not turn the whole park pale.
          solid: 0x5b239b,
          emissive: 0x250c49,
          line: 0xc58ee8,
          opacity: 1,
          wireOpacity: 0.26,
          lineOpacity: 0.46,
        },
        vegetation: {
          solid: 0x07382d,
          emissive: 0x031f1a,
          line: 0x54cfc3,
          opacity: 1,
          wireOpacity: 0.16,
          lineOpacity: 0.3,
        },
        tse: {
          solid: 0x1558ba,
          emissive: 0x061f52,
          line: 0x66afe6,
          opacity: 1,
          wireOpacity: 0.24,
          lineOpacity: 0.44,
        },
      };
      return palettes[layerKey] || palettes.structures;
    },

    getStandaloneFeaturePalette(layerKey, featureType = "", properties = {}) {
      const base = this.getStandaloneLayerPalette(layerKey);
      const type = String(featureType || "").toLowerCase();
      const remarks = String(
        properties.QC_REMARKS ||
          properties.Remarks ||
          properties.DC_REMARK ||
          properties.species ||
          "",
      ).toLowerCase();
      const description = `${type} ${remarks}`;

      if (layerKey === "water") {
        return {
          key: "water",
          solid: 0x075b8f,
          emissive: 0x063c68,
          line: 0x69ecff,
          opacity: 0.68,
        };
      }

      if (layerKey === "paths") {
        const pathDescription = [
          type,
          properties.CL_LAND_COVER_DETAILED,
          properties.CL_LANDUSE_DETAILED,
          properties.CL_ROAD_TYPE,
          properties.ROAD_TYPE,
          properties.Road_Type,
          properties.road_type,
          properties.sourceLayer,
          properties.source_layer,
          properties._source_layer,
          properties.layerName,
          properties.layer_name,
          properties.category,
          properties.subtype,
          properties.Type,
          properties.type,
          properties.Remarks,
          properties.QC_REMARKS,
          properties.DC_REMARK,
        ]
          .filter((value) => value != null && String(value).trim())
          .join(" ")
          .toLowerCase()
          .replace(/[_-]+/g, " ");

        // A Road_Section source name can contain both Road and Sidewalk
        // features. Classify the specific feature value first, then fall back
        // to the broader source/layer description.
        const isSidewalkSurface = /\b(sidewalk|foot\s*path|footpath)\b/.test(
          pathDescription,
        );
        const isPedestrianSurface =
          /\b(pedestrian(?:\s+space)?|walk\s*way|walkway|walk\s*path|walking|cycle\s*path|cycleway|trail|stair|stairs|staircase|ramp|footbridge)\b/.test(
            pathDescription,
          );
        const isRoadOrParkingSurface =
          /\b(car\s*parking|car\s*park|carpark|parking(?:\s+(?:lot|area))?|carriageway|roadway|vehicular|vehicle\s*access|driveway|street|asphalt|road)\b/.test(
            pathDescription,
          );

        if (isSidewalkSurface) {
          return {
            key: "path-sidewalk",
            solid: THREE_PATH_SIDEWALK_COLOR,
            emissive: THREE_PATH_SIDEWALK_EMISSIVE,
            line: THREE_PATH_SIDEWALK_LINE_COLOR,
            opacity: 1,
            wireOpacity: 0.1,
            lineOpacity: 0.24,
          };
        }

        if (isPedestrianSurface) {
          return {
            key: "path-pedestrian",
            solid: THREE_PATH_PEDESTRIAN_COLOR,
            emissive: THREE_PATH_PEDESTRIAN_EMISSIVE,
            line: THREE_PATH_PEDESTRIAN_LINE_COLOR,
            opacity: 1,
            wireOpacity: 0.08,
            lineOpacity: 0.2,
          };
        }

        if (isRoadOrParkingSurface) {
          return {
            key: "path-road-parking",
            solid: THREE_PATH_ROAD_PARKING_COLOR,
            emissive: THREE_PATH_ROAD_PARKING_EMISSIVE,
            line: THREE_PATH_ROAD_PARKING_LINE_COLOR,
            opacity: 1,
            wireOpacity: 0.12,
            lineOpacity: 0.28,
          };
        }

        return { key: "path-other", ...base };
      }

      if (layerKey !== "vegetation") return { key: layerKey, ...base };

      if (/shrub/.test(type)) {
        if (/atriplex|haloxylon|saltbush/.test(description)) {
          return {
            key: "shrub-saltbush",
            solid: 0x5d775f,
            emissive: 0x17271b,
            line: 0x78927b,
            opacity: 1,
            wireOpacity: 0.04,
            lineOpacity: 0.08,
          };
        }

        if (/caligonum|capparis/.test(description)) {
          return {
            key: "shrub-desert-olive",
            solid: 0x657048,
            emissive: 0x1e2614,
            line: 0x82905e,
            opacity: 1,
            wireOpacity: 0.04,
            lineOpacity: 0.08,
          };
        }

        if (/crinum/.test(description)) {
          return {
            key: "shrub-crinum",
            solid: 0x4e784b,
            emissive: 0x142514,
            line: 0x71a06d,
            opacity: 1,
            wireOpacity: 0.035,
            lineOpacity: 0.07,
          };
        }

        if (/boug(?:ain|an)villea|senecio/.test(description)) {
          return {
            key: "shrub-flowering",
            solid: 0x42683d,
            emissive: 0x142713,
            line: 0x699164,
            opacity: 1,
            wireOpacity: 0.04,
            lineOpacity: 0.08,
          };
        }

        if (
          /clerodendrum|dodonea|vitex|jasminum|quisqualis|quiscualis/.test(
            description,
          )
        ) {
          return {
            key: "shrub-broadleaf",
            solid: 0x365f39,
            emissive: 0x102511,
            line: 0x5c875f,
            opacity: 1,
            wireOpacity: 0.04,
            lineOpacity: 0.08,
          };
        }

        return {
          key: "shrub-natural",
          solid: 0x486a43,
          emissive: 0x142615,
          line: 0x6d9167,
          opacity: 1,
          wireOpacity: 0.04,
          lineOpacity: 0.08,
        };
      }

      if (/ground[_ -]?cover/.test(type)) {
        if (
          /pen+isetum.*rubrum|rubrum|red[_ -]?fountain|purple/.test(description)
        ) {
          return {
            key: "groundcover-pennisetum-rubrum",
            solid: 0x713c55,
            emissive: 0x24131c,
            line: 0x87566c,
            opacity: 1,
            wireOpacity: 0.02,
            lineOpacity: 0.055,
          };
        }

        if (/pen+isetum|fountain grass/.test(description)) {
          return {
            key: "groundcover-pennisetum",
            solid: 0x687b4d,
            emissive: 0x202816,
            line: 0x81935f,
            opacity: 1,
            wireOpacity: 0.02,
            lineOpacity: 0.055,
          };
        }

        if (/lawn|turf/.test(description)) {
          return {
            key: "groundcover-lawn",
            solid: 0x3f7d3f,
            emissive: 0x132b14,
            line: 0x5b965a,
            opacity: 1,
            wireOpacity: 0.018,
            lineOpacity: 0.05,
          };
        }

        if (/sporobolus/.test(description)) {
          return {
            key: "groundcover-sporobolus",
            solid: 0x5e8069,
            emissive: 0x1a2921,
            line: 0x769786,
            opacity: 1,
            wireOpacity: 0.02,
            lineOpacity: 0.055,
          };
        }

        if (/cenchrus/.test(description)) {
          return {
            key: "groundcover-cenchrus",
            solid: 0x697941,
            emissive: 0x202715,
            line: 0x829156,
            opacity: 1,
            wireOpacity: 0.02,
            lineOpacity: 0.055,
          };
        }

        if (/aristida/.test(description)) {
          return {
            key: "groundcover-aristida",
            solid: 0x7d8153,
            emissive: 0x292a19,
            line: 0x969769,
            opacity: 1,
            wireOpacity: 0.02,
            lineOpacity: 0.055,
          };
        }

        if (/lausirius|lasiurus|lasirus/.test(description)) {
          return {
            key: "groundcover-lasiurus",
            solid: 0x8b8252,
            emissive: 0x2d2918,
            line: 0xa09869,
            opacity: 1,
            wireOpacity: 0.02,
            lineOpacity: 0.055,
          };
        }

        if (/stipagrostis/.test(description)) {
          return {
            key: "groundcover-stipagrostis",
            solid: 0x8c9873,
            emissive: 0x2b3022,
            line: 0xa5ae8d,
            opacity: 1,
            wireOpacity: 0.02,
            lineOpacity: 0.055,
          };
        }

        return {
          key: "groundcover-natural-grass",
          solid: 0x65774b,
          emissive: 0x1e2717,
          line: 0x7f9162,
          opacity: 1,
          wireOpacity: 0.02,
          lineOpacity: 0.055,
        };
      }

      return {
        key: "vegetation-default",
        solid: 0x3f6844,
        emissive: 0x132816,
        line: 0x648b68,
        opacity: 1,
        wireOpacity: 0.05,
        lineOpacity: 0.1,
      };
    },

    createStandaloneLayerMaterials(layerKey, paletteOverride = null) {
      const palette =
        paletteOverride || this.getStandaloneLayerPalette(layerKey);
      const paletteKey = String((palette && palette.key) || "").toLowerCase();
      const isGroundcoverPalette =
        layerKey === "vegetation" && /^groundcover/.test(paletteKey);
      const requestedWireOpacity = Number(palette && palette.wireOpacity);
      const requestedLineOpacity = Number(palette && palette.lineOpacity);
      const polygonOffsetEnabled = ["paths", "water", "vegetation"].includes(
        layerKey,
      );
      const polygonOffsetFactor =
        layerKey === "water" ? -18 : layerKey === "vegetation" ? -4 : -1;
      const polygonOffsetUnits =
        layerKey === "water" ? -36 : layerKey === "vegetation" ? -8 : -2;
      const commonSurfaceProperties = {
        transparent: false,
        opacity: 1,
        side: THREE.DoubleSide,
        depthWrite: true,
        depthTest: true,
        polygonOffset: polygonOffsetEnabled,
        polygonOffsetFactor,
        polygonOffsetUnits,
      };
      const useUnlitTopsideColour =
        layerKey === "water" ||
        layerKey === "paths" ||
        (layerKey === "vegetation" && !isGroundcoverPalette);
      let solid;

      if (isGroundcoverPalette) {
        // GroundCover is grass in this dataset. A fully rough, light-reactive
        // material gives the species-coloured areas a natural planted surface
        // instead of the flat cyan/green technical-fill appearance.
        solid = new THREE.MeshStandardMaterial({
          color: palette.solid,
          emissive: palette.emissive,
          emissiveIntensity: 0.16,
          roughness: 1,
          metalness: 0,
          flatShading: false,
          ...commonSurfaceProperties,
        });
      } else if (useUnlitTopsideColour) {
        solid = new THREE.MeshBasicMaterial({
          // Every GIS object fill is fully opaque. The technical wireframe
          // remains a separate overlay, but the object itself can no longer
          // be seen through.
          color: palette.solid,
          ...commonSurfaceProperties,
        });

        if (layerKey === "paths" && paletteKey === "path-pedestrian") {
          // Add a regular world-space square grid to the actual draped
          // PEDESTRIAN SPACE surface only. Roads, Sidewalk and Car Parking
          // remain visually clean. Because this is evaluated on the path
          // mesh itself, the grid rises/falls with Terrain (after) instead of
          // floating flat.
          const pathGridColor = new THREE.Color(palette.line);
          solid.onBeforeCompile = (shader) => {
            shader.uniforms.uPathGridColor = { value: pathGridColor };
            shader.uniforms.uPathGridCell = {
              value: THREE_PATH_GRID_CELL_METERS,
            };
            shader.uniforms.uPathGridOpacity = {
              value: THREE_PATH_GRID_OPACITY,
            };

            shader.vertexShader = shader.vertexShader
              .replace(
                "#include <common>",
                `#include <common>
varying vec2 vPathGridWorldXZ;`,
              )
              .replace(
                "#include <begin_vertex>",
                `#include <begin_vertex>
vec4 pathGridWorldPosition = modelMatrix * vec4(transformed, 1.0);
vPathGridWorldXZ = pathGridWorldPosition.xz;`,
              );

            shader.fragmentShader = shader.fragmentShader
              .replace(
                "#include <common>",
                `#include <common>
varying vec2 vPathGridWorldXZ;
uniform vec3 uPathGridColor;
uniform float uPathGridCell;
uniform float uPathGridOpacity;`,
              )
              .replace(
                "#include <opaque_fragment>",
                `vec2 pathGridUv = vPathGridWorldXZ / max(uPathGridCell, 0.001);
vec2 pathGridFw = max(fwidth(pathGridUv), vec2(0.0001));
vec2 pathGridDist =
  abs(fract(pathGridUv - 0.5) - 0.5) /
  (pathGridFw * 0.48);
float pathGridLine = 1.0 - min(min(pathGridDist.x, pathGridDist.y), 1.0);
pathGridLine = smoothstep(0.06, 0.98, pathGridLine);
diffuseColor.rgb = mix(
  diffuseColor.rgb,
  uPathGridColor,
  pathGridLine * uPathGridOpacity
);
#include <opaque_fragment>`,
              );
          };
          solid.customProgramCacheKey = () =>
            `path-elevation-grid-${paletteKey || "default"}-${THREE_PATH_GRID_CELL_METERS}-${THREE_PATH_GRID_OPACITY}`;
        }
      } else {
        solid = new THREE.MeshStandardMaterial({
          color: palette.solid,
          emissive: palette.emissive,
          emissiveIntensity: 0.5,
          roughness: 0.82,
          metalness: 0.04,
          ...commonSurfaceProperties,
        });
      }

      return {
        solid,
        // Wire and line overlays may use a lighter alpha for legibility, but
        // they no longer control the opacity of the solid 3D object beneath.
        wire: new THREE.MeshBasicMaterial({
          color: palette.line,
          wireframe: true,
          transparent: true,
          opacity: Number.isFinite(requestedWireOpacity)
            ? requestedWireOpacity
            : layerKey === "water"
              ? 0.035
              : layerKey === "paths"
                ? 0.24
                : isGroundcoverPalette
                  ? 0.18
                  : 0.42,
          depthWrite: false,
          blending:
            layerKey === "water"
              ? THREE.NormalBlending
              : THREE.AdditiveBlending,
        }),
        lines: new THREE.LineBasicMaterial({
          color: palette.line,
          transparent: true,
          opacity: Number.isFinite(requestedLineOpacity)
            ? requestedLineOpacity
            : layerKey === "water"
              ? 0.08
              : layerKey === "paths"
                ? 0.46
                : isGroundcoverPalette
                  ? 0.24
                  : 0.56,
          depthWrite: false,
        }),
        glow: new THREE.MeshBasicMaterial({
          color: palette.line,
          transparent: true,
          opacity: 0.82,
          depthWrite: false,
          blending: THREE.AdditiveBlending,
        }),
      };
    },

    createStandalonePergolaMaterials() {
      return {
        solid: new THREE.MeshStandardMaterial({
          color: THREE_PARK_PERGOLA_SOLID_COLOR,
          emissive: THREE_PARK_PERGOLA_EMISSIVE_COLOR,
          emissiveIntensity: 0.34,
          roughness: 0.9,
          metalness: 0.02,
          transparent: false,
          opacity: 1,
          side: THREE.DoubleSide,
          depthWrite: true,
          depthTest: true,
        }),
        lines: new THREE.LineBasicMaterial({
          color: THREE_PARK_PERGOLA_LINE_COLOR,
          transparent: true,
          opacity: 0.82,
          depthWrite: false,
        }),
      };
    },

    makeStandaloneLayerGroupOpaque(group) {
      if (!group || typeof group.traverse !== "function") return;

      group.traverse((object) => {
        if (!object) return;
        const objectName = String(object.name || "").toLowerCase();

        // Light bulbs, halos, and ground pools are deliberately translucent
        // additive effects, not physical GIS geometry. Keep these effects on
        // so an illuminated light pole is immediately distinguishable from a
        // plain utility pole. Other decorative layer glows remain hidden to
        // preserve the fully opaque map-object style.
        const isLightPoleGlowEffect =
          objectName === "structure-light-bulbs" ||
          objectName === "structure-light-inner-halos" ||
          objectName === "structure-light-outer-halos" ||
          objectName === "structure-light-ground-pools";
        if (isLightPoleGlowEffect) {
          object.visible = true;
          return;
        }

        if (
          /(?:^|[-_])(glow|halo|halos)(?:$|[-_])/.test(objectName) ||
          objectName.includes("ground-pools")
        ) {
          object.visible = false;
          return;
        }

        if (!object.material) return;
        const materials = Array.isArray(object.material)
          ? object.material
          : [object.material];

        materials.forEach((material) => {
          if (!material || !material.isMaterial) return;

          // Solid mesh fills must be completely opaque. Keep wireframe and line
          // overlays separate because their gaps are intentional drawing detail,
          // not object transparency.
          const isSolidMeshMaterial =
            Boolean(object.isMesh) && !Boolean(material.wireframe);
          if (!isSolidMeshMaterial) return;

          material.transparent = false;
          material.opacity = 1;
          material.depthWrite = true;
          material.depthTest = true;
          material.blending = THREE.NormalBlending;
          material.needsUpdate = true;
        });
      });
    },

    appendStandaloneFlatPolygonGeometry(
      localRings,
      modelData,
      heightOffset,
      positions,
      linePositions,
      surfaceOptions = false,
    ) {
      if (!localRings.length) return;
      const outer = localRings[0];
      const holes = localRings.slice(1);
      const vertices = outer.concat(...holes);
      const faces = THREE.ShapeUtils.triangulateShape(outer, holes);
      const options =
        surfaceOptions && typeof surfaceOptions === "object"
          ? surfaceOptions
          : {
              preventTerrainOcclusion: Boolean(surfaceOptions),
            };
      const preventTerrainOcclusion = Boolean(options.preventTerrainOcclusion);
      const drapeToTerrain = Boolean(options.drapeToTerrain);
      const terrainClearance = Math.max(
        0,
        Number(options.terrainClearance) || 0.12,
      );
      const drapeSegmentLength = Math.max(
        1.25,
        Number(options.drapeSegmentLength) || THREE_PATH_DRAPE_SEGMENT_METERS,
      );
      const maxDrapeSubdivisions = Math.max(
        1,
        Math.min(
          32,
          Number(options.maxDrapeSubdivisions) ||
            THREE_PATH_DRAPE_MAX_SUBDIVISIONS,
        ),
      );
      const heightCache = new Map();
      const getDrapedHeight = (point) => {
        const x = Number(point && point.x);
        const z = Number(point && point.y);
        if (!Number.isFinite(x) || !Number.isFinite(z)) return heightOffset;
        // Quantize to centimetres rather than millimetres. Generated drape vertices
        // often land extremely close to one another, and sharing those samples
        // substantially reduces repeated terrain-height lookups.
        const key = `${Math.round(x * 100)}|${Math.round(z * 100)}`;
        if (heightCache.has(key)) return heightCache.get(key);
        const terrainHeight = this.getStandaloneTerrainHeightAtPoint(
          { x, z },
          modelData,
        );
        const value =
          (Number.isFinite(Number(terrainHeight)) ? Number(terrainHeight) : 0) +
          heightOffset;
        heightCache.set(key, value);
        return value;
      };
      const interpolateTrianglePoint = (
        pointA,
        pointB,
        pointC,
        stepB,
        stepC,
        divisions,
      ) => {
        const weightB = stepB / divisions;
        const weightC = stepC / divisions;
        const weightA = 1 - weightB - weightC;
        return new THREE.Vector2(
          pointA.x * weightA + pointB.x * weightB + pointC.x * weightC,
          pointA.y * weightA + pointB.y * weightB + pointC.y * weightC,
        );
      };
      const appendDrapedTriangle = (pointA, pointB, pointC) => {
        [pointA, pointB, pointC].forEach((point) => {
          positions.push(point.x, getDrapedHeight(point), point.y);
        });
      };

      if (drapeToTerrain && faces.length) {
        // Walking paths in the current datasets are all on-grade. Split each
        // source triangle into a bounded grid and sample Terrain (after) at
        // every generated vertex. This prevents a large path triangle from
        // bridging across a slope and covering buildings, poles, or pergolas.
        faces.forEach((face) => {
          const pointA = vertices[face[0]];
          const pointB = vertices[face[1]];
          const pointC = vertices[face[2]];
          if (!pointA || !pointB || !pointC) return;

          const maximumEdge = Math.max(
            pointA.distanceTo(pointB),
            pointB.distanceTo(pointC),
            pointC.distanceTo(pointA),
          );
          const divisions = Math.max(
            1,
            Math.min(
              maxDrapeSubdivisions,
              Math.ceil(maximumEdge / drapeSegmentLength),
            ),
          );

          for (let stepB = 0; stepB < divisions; stepB += 1) {
            for (let stepC = 0; stepC < divisions - stepB; stepC += 1) {
              const lowerLeft = interpolateTrianglePoint(
                pointA,
                pointB,
                pointC,
                stepB,
                stepC,
                divisions,
              );
              const lowerRight = interpolateTrianglePoint(
                pointA,
                pointB,
                pointC,
                stepB + 1,
                stepC,
                divisions,
              );
              const upperLeft = interpolateTrianglePoint(
                pointA,
                pointB,
                pointC,
                stepB,
                stepC + 1,
                divisions,
              );
              appendDrapedTriangle(lowerLeft, lowerRight, upperLeft);

              if (stepB + stepC <= divisions - 2) {
                const upperRight = interpolateTrianglePoint(
                  pointA,
                  pointB,
                  pointC,
                  stepB + 1,
                  stepC + 1,
                  divisions,
                );
                appendDrapedTriangle(lowerRight, upperRight, upperLeft);
              }
            }
          }
        });
      } else {
        const heights = vertices.map((point) => getDrapedHeight(point));
        const vertexLift = new Array(vertices.length).fill(0);

        if (preventTerrainOcclusion && faces.length) {
          // A large flat GIS triangle can cross a Terrain (after) ridge even
          // when all boundary vertices are correctly draped. Non-path layers
          // keep the bounded face-lift fallback to avoid expensive tessellation.
          faces.forEach((face) => {
            const [indexA, indexB, indexC] = face;
            const pointA = vertices[indexA];
            const pointB = vertices[indexB];
            const pointC = vertices[indexC];
            if (!pointA || !pointB || !pointC) return;

            const edgeAB = pointA.distanceTo(pointB);
            const edgeBC = pointB.distanceTo(pointC);
            const edgeCA = pointC.distanceTo(pointA);
            const maximumEdge = Math.max(edgeAB, edgeBC, edgeCA);
            const divisions = Math.max(
              2,
              Math.min(8, Math.ceil(maximumEdge / 5.5)),
            );
            let faceLift = 0;

            for (let stepA = 0; stepA <= divisions; stepA += 1) {
              for (let stepB = 0; stepB <= divisions - stepA; stepB += 1) {
                const weightA = stepA / divisions;
                const weightB = stepB / divisions;
                const weightC = 1 - weightA - weightB;
                const sampleX =
                  pointA.x * weightA + pointB.x * weightB + pointC.x * weightC;
                const sampleZ =
                  pointA.y * weightA + pointB.y * weightB + pointC.y * weightC;
                const terrainY = this.getStandaloneTerrainHeightAtPoint(
                  { x: sampleX, z: sampleZ },
                  modelData,
                );
                const currentSurfaceY =
                  heights[indexA] * weightA +
                  heights[indexB] * weightB +
                  heights[indexC] * weightC;
                const requiredLift =
                  terrainY + heightOffset + terrainClearance - currentSurfaceY;

                if (Number.isFinite(requiredLift)) {
                  faceLift = Math.max(faceLift, requiredLift);
                }
              }
            }

            faceLift = Math.max(0, faceLift);
            face.forEach((vertexIndex) => {
              vertexLift[vertexIndex] = Math.max(
                vertexLift[vertexIndex],
                faceLift,
              );
            });
          });
        }

        faces.forEach((face) => {
          face.forEach((vertexIndex) => {
            const point = vertices[vertexIndex];
            positions.push(
              point.x,
              heights[vertexIndex] + vertexLift[vertexIndex],
              point.y,
            );
          });
        });
      }

      // Densify path outlines as well, otherwise a long two-point outline can
      // still visually bridge across a slope even when the fill is draped.
      localRings.forEach((ring) => {
        ring.forEach((point, index) => {
          const next = ring[(index + 1) % ring.length];
          const edgeLength = point.distanceTo(next);
          const outlineSegmentLength = drapeToTerrain
            ? Math.max(drapeSegmentLength, THREE_PATH_OUTLINE_SEGMENT_METERS)
            : drapeSegmentLength;
          const divisions = drapeToTerrain
            ? Math.max(
                1,
                Math.min(32, Math.ceil(edgeLength / outlineSegmentLength)),
              )
            : 1;

          for (let step = 0; step < divisions; step += 1) {
            const startRatio = step / divisions;
            const endRatio = (step + 1) / divisions;
            const segmentStart = new THREE.Vector2(
              THREE.MathUtils.lerp(point.x, next.x, startRatio),
              THREE.MathUtils.lerp(point.y, next.y, startRatio),
            );
            const segmentEnd = new THREE.Vector2(
              THREE.MathUtils.lerp(point.x, next.x, endRatio),
              THREE.MathUtils.lerp(point.y, next.y, endRatio),
            );
            linePositions.push(
              segmentStart.x,
              getDrapedHeight(segmentStart) + 0.015,
              segmentStart.y,
              segmentEnd.x,
              getDrapedHeight(segmentEnd) + 0.015,
              segmentEnd.y,
            );
          }
        });
      });
    },

    appendStandaloneExtrudedPolygonGeometry(
      localRings,
      modelData,
      height,
      positions,
      linePositions,
    ) {
      if (!localRings.length) return;
      const outer = localRings[0];
      const holes = localRings.slice(1);
      const vertices = outer.concat(...holes);
      const faces = THREE.ShapeUtils.triangulateShape(outer, holes);
      const baseHeights = vertices.map(
        (point) =>
          this.getStandaloneTerrainHeightAtPoint(
            { x: point.x, z: point.y },
            modelData,
          ) + THREE_STRUCTURE_SURFACE_CLEARANCE_METERS,
      );
      const roofY = Math.max(...baseHeights) + Math.max(0.12, height);

      faces.forEach((face) => {
        face.forEach((vertexIndex) => {
          const point = vertices[vertexIndex];
          positions.push(point.x, roofY, point.y);
        });
      });

      localRings.forEach((ring) => {
        ring.forEach((point, index) => {
          const next = ring[(index + 1) % ring.length];
          const baseA =
            this.getStandaloneTerrainHeightAtPoint(
              { x: point.x, z: point.y },
              modelData,
            ) + THREE_STRUCTURE_SURFACE_CLEARANCE_METERS;
          const baseB =
            this.getStandaloneTerrainHeightAtPoint(
              { x: next.x, z: next.y },
              modelData,
            ) + THREE_STRUCTURE_SURFACE_CLEARANCE_METERS;

          positions.push(
            point.x,
            roofY,
            point.y,
            point.x,
            baseA,
            point.y,
            next.x,
            baseB,
            next.y,

            point.x,
            roofY,
            point.y,
            next.x,
            baseB,
            next.y,
            next.x,
            roofY,
            next.y,
          );

          linePositions.push(
            point.x,
            roofY + 0.015,
            point.y,
            next.x,
            roofY + 0.015,
            next.y,
            point.x,
            baseA,
            point.y,
            point.x,
            roofY,
            point.y,
          );
        });
      });
    },

    isStandaloneCoveredPergolaStructure(featureType = "", properties = {}) {
      const searchableText = [
        featureType,
        properties.QC_REMARKS,
        properties.APP_MODEL,
        properties.app_model,
        properties.APP_3D_MODEL,
        properties.app_3d_model,
        properties["App 3D Model"],
        properties.APP_DISPLAY_LABEL,
        properties["App Display Label"],
        properties.Remarks,
        properties.Type,
        properties.type,
        properties.name,
        properties.name_en,
        properties.NAME,
        properties.label,
        properties.category,
        properties.subtype,
        properties.sourceLayer,
      ]
        .filter((value) => value != null && String(value).trim())
        .join(" ")
        .toLowerCase()
        .replace(/[_-]+/g, " ");

      return /\bcovered\s+pergola\b/.test(searchableText);
    },

    getStandalonePolygonCentroid(ring = []) {
      if (!Array.isArray(ring) || ring.length < 3) return null;

      let areaAccumulator = 0;
      let centroidX = 0;
      let centroidY = 0;

      ring.forEach((point, index) => {
        const next = ring[(index + 1) % ring.length];
        const cross = point.x * next.y - next.x * point.y;
        areaAccumulator += cross;
        centroidX += (point.x + next.x) * cross;
        centroidY += (point.y + next.y) * cross;
      });

      const area = areaAccumulator / 2;
      if (Math.abs(area) < 0.000001) {
        const average = ring.reduce(
          (accumulator, point) => {
            accumulator.x += Number(point.x) || 0;
            accumulator.y += Number(point.y) || 0;
            return accumulator;
          },
          { x: 0, y: 0 },
        );

        return new THREE.Vector2(
          average.x / ring.length,
          average.y / ring.length,
        );
      }

      return new THREE.Vector2(centroidX / (6 * area), centroidY / (6 * area));
    },

    getStandalonePergolaSupportPoints(ring = []) {
      if (!Array.isArray(ring) || !ring.length) return [];

      const candidates = [
        ring.reduce(
          (best, point) => (!best || point.x < best.x ? point : best),
          null,
        ),
        ring.reduce(
          (best, point) => (!best || point.x > best.x ? point : best),
          null,
        ),
        ring.reduce(
          (best, point) => (!best || point.y < best.y ? point : best),
          null,
        ),
        ring.reduce(
          (best, point) => (!best || point.y > best.y ? point : best),
          null,
        ),
      ].filter(Boolean);

      const unique = [];
      candidates.forEach((point) => {
        const exists = unique.some(
          (existing) =>
            Math.abs(existing.x - point.x) < 0.05 &&
            Math.abs(existing.y - point.y) < 0.05,
        );
        if (!exists) unique.push(point);
      });

      return unique;
    },

    getStandalonePolygonBounds(ring = []) {
      if (!Array.isArray(ring) || !ring.length) return null;

      const xValues = ring
        .map((point) => Number(point.x))
        .filter(Number.isFinite);
      const yValues = ring
        .map((point) => Number(point.y))
        .filter(Number.isFinite);
      if (!xValues.length || !yValues.length) return null;

      return {
        minX: Math.min(...xValues),
        maxX: Math.max(...xValues),
        minY: Math.min(...yValues),
        maxY: Math.max(...yValues),
      };
    },

    doStandalonePolygonBoundsOverlap(first, second, padding = 0.35) {
      if (!first || !second) return false;
      return !(
        first.maxX < second.minX - padding ||
        first.minX > second.maxX + padding ||
        first.maxY < second.minY - padding ||
        first.minY > second.maxY + padding
      );
    },

    getStandalonePergolaRequiredClearance(
      pergolaRings,
      structureFootprints = [],
      lightPoleRecords = [],
    ) {
      const pergolaBounds = this.getStandalonePolygonBounds(pergolaRings?.[0]);
      // Keep the architectural pergola above the nearby building roofs without
      // recreating the excessive canopy height from the earlier version.
      // A 4.4 m minimum keeps the detailed pergola comfortably above nearby
      // buildings and lighting while preserving a human-scale canopy.
      let requiredClearance = 4.4;

      structureFootprints.forEach((structure) => {
        if (!structure || structure.isPergola) return;
        if (
          !this.doStandalonePolygonBoundsOverlap(
            pergolaBounds,
            structure.bounds,
          )
        ) {
          return;
        }

        // Where a pergola genuinely overlaps a building, leave a visible but
        // still modest roof-to-canopy separation. This keeps the canopy high
        // enough around lighting while avoiding the old several-metre gap.
        requiredClearance = Math.max(
          requiredClearance,
          Number(structure.height || 0) + 1.4,
        );
      });

      // A light pole that falls directly beneath/at the edge of the pergola
      // gets a limited clearance adjustment. We deliberately cap this so a
      // 5-6 m pole cannot lift the entire architectural canopy excessively.
      if (pergolaBounds) {
        lightPoleRecords.forEach((pole) => {
          const point = pole && pole.local;
          if (!point) return;
          const x = Number(point.x);
          const y = Number(point.z);
          if (!Number.isFinite(x) || !Number.isFinite(y)) return;
          const margin = 0.8;
          const withinPergola =
            x >= pergolaBounds.minX - margin &&
            x <= pergolaBounds.maxX + margin &&
            y >= pergolaBounds.minY - margin &&
            y <= pergolaBounds.maxY + margin;
          if (!withinPergola) return;

          const poleHeight = Number(pole.height || 0);
          if (poleHeight > 0) {
            requiredClearance = Math.max(
              requiredClearance,
              Math.min(5.2, poleHeight - 0.25),
            );
          }
        });
      }

      return requiredClearance;
    },

    getStandalonePergolaModelNumber(properties = {}) {
      const rawModel = String(
        this.getStandaloneCaseInsensitiveProperty(properties, [
          "APP_MODEL",
          "app_model",
          "APP_3D_MODEL",
          "app_3d_model",
          "App 3D Model",
          "MODEL",
          "model",
          "Pergola_Model",
          "pergola_model",
        ]) || "",
      )
        .trim()
        .toUpperCase();

      const coveredPergolaMatch = rawModel.match(/COVERED[_\s-]*PERGOLA[_\s-]*0?([12])\b/i);
      if (coveredPergolaMatch) {
        return Number(coveredPergolaMatch[1]) === 1 ? 101 : 102;
      }

      const match = rawModel.match(/(?:QAD|CAD)[_\s-]*0?(\d{1,2})/i);
      if (!match) return null;

      const modelNumber = Number(match[1]);
      return modelNumber >= 1 && modelNumber <= 9 ? modelNumber : null;
    },

    getStandalonePergolaDesignProfile(modelNumber) {
      // The consultant roof-canopy drawings are based on a repeating 9.6 m
      // square structural module. The GIS footprints are exact unions of those
      // modules (for example QAD_05 = 11 modules and QAD_08 = 5 modules), so
      // rebuilding the module grid from the polygon keeps each canopy aligned
      // to its surveyed footprint instead of replacing it with a generic roof.
      // QAD_09 has no supplied detail sheet; its GIS footprint is the same 2x2
      // four-module square as QAD_04, so QAD_04 is intentionally reused.
      const profiles = {
        1: { moduleCount: 6, phase: 0, foldPhase: 0, referenceModel: 1 },
        2: { moduleCount: 10, phase: 1, foldPhase: 0, referenceModel: 2 },
        3: { moduleCount: 4, phase: 1, foldPhase: 1, referenceModel: 3 },
        4: { moduleCount: 4, phase: 0, foldPhase: 0, referenceModel: 4 },
        5: { moduleCount: 11, phase: 0, foldPhase: 1, referenceModel: 5 },
        6: { moduleCount: 4, phase: 1, foldPhase: 0, referenceModel: 6 },
        7: { moduleCount: 4, phase: 0, foldPhase: 1, referenceModel: 7 },
        8: { moduleCount: 5, phase: 1, foldPhase: 1, referenceModel: 8 },
        9: { moduleCount: 4, phase: 0, foldPhase: 0, referenceModel: 4 },
        // Al Monasya roof-canopy sheets P01/P02. Both are four-module
        // covered pergolas, but their folded high/low roof arrangement is
        // mirrored, so keep them as two distinct procedural profiles.
        101: { moduleCount: 4, phase: 0, foldPhase: 0, referenceModel: "P01" },
        102: { moduleCount: 4, phase: 1, foldPhase: 1, referenceModel: "P02" },
      };

      return profiles[Number(modelNumber)] || null;
    },

    appendStandaloneGenericPergolaCanopyGeometry(
      localRings,
      modelData,
      height,
      positions,
      linePositions,
    ) {
      if (!localRings.length) return;

      const outer = localRings[0];
      if (!outer || outer.length < 3) {
        this.appendStandaloneExtrudedPolygonGeometry(
          localRings,
          modelData,
          height,
          positions,
          linePositions,
        );
        return;
      }

      const centroid = this.getStandalonePolygonCentroid(outer);
      if (!centroid) {
        this.appendStandaloneExtrudedPolygonGeometry(
          localRings,
          modelData,
          height,
          positions,
          linePositions,
        );
        return;
      }

      const supportClearance = Math.max(4.4, Number(height) || 0);
      const canopyRise = Math.max(0.75, Math.min(1.8, supportClearance * 0.18));
      const centroidTerrainY =
        this.getStandaloneTerrainHeightAtPoint(
          { x: centroid.x, z: centroid.y },
          modelData,
        ) + THREE_STRUCTURE_SURFACE_CLEARANCE_METERS;
      const peakY = centroidTerrainY + supportClearance + canopyRise;

      outer.forEach((point, index) => {
        const next = outer[(index + 1) % outer.length];
        const edgeBaseA =
          this.getStandaloneTerrainHeightAtPoint(
            { x: point.x, z: point.y },
            modelData,
          ) + THREE_STRUCTURE_SURFACE_CLEARANCE_METERS;
        const edgeBaseB =
          this.getStandaloneTerrainHeightAtPoint(
            { x: next.x, z: next.y },
            modelData,
          ) + THREE_STRUCTURE_SURFACE_CLEARANCE_METERS;
        const eaveYA = edgeBaseA + supportClearance;
        const eaveYB = edgeBaseB + supportClearance;

        positions.push(
          centroid.x,
          peakY,
          centroid.y,
          point.x,
          eaveYA,
          point.y,
          next.x,
          eaveYB,
          next.y,
        );

        linePositions.push(
          point.x,
          eaveYA + 0.015,
          point.y,
          next.x,
          eaveYB + 0.015,
          next.y,
          centroid.x,
          peakY + 0.015,
          centroid.y,
          point.x,
          eaveYA + 0.015,
          point.y,
        );
      });

      this.getStandalonePergolaSupportPoints(outer).forEach((point) => {
        const baseY =
          this.getStandaloneTerrainHeightAtPoint(
            { x: point.x, z: point.y },
            modelData,
          ) + THREE_STRUCTURE_SURFACE_CLEARANCE_METERS;
        const topY = baseY + supportClearance;
        const halfWidth = 0.07;

        positions.push(
          point.x - halfWidth,
          topY,
          point.y,
          point.x - halfWidth,
          baseY,
          point.y,
          point.x + halfWidth,
          baseY,
          point.y,

          point.x - halfWidth,
          topY,
          point.y,
          point.x + halfWidth,
          baseY,
          point.y,
          point.x + halfWidth,
          topY,
          point.y,

          point.x,
          topY,
          point.y - halfWidth,
          point.x,
          baseY,
          point.y - halfWidth,
          point.x,
          baseY,
          point.y + halfWidth,

          point.x,
          topY,
          point.y - halfWidth,
          point.x,
          baseY,
          point.y + halfWidth,
          point.x,
          topY,
          point.y + halfWidth,
        );

        linePositions.push(
          point.x,
          baseY,
          point.y,
          point.x,
          topY + 0.015,
          point.y,
        );
      });
    },

    appendStandalonePergolaCanopyGeometry(
      localRings,
      modelData,
      height,
      positions,
      linePositions,
      properties = {},
    ) {
      if (!localRings.length) return;

      const outer = localRings[0];
      if (!outer || outer.length < 3) {
        this.appendStandaloneGenericPergolaCanopyGeometry(
          localRings,
          modelData,
          height,
          positions,
          linePositions,
        );
        return;
      }

      const modelNumber = this.getStandalonePergolaModelNumber(properties);
      const designProfile = this.getStandalonePergolaDesignProfile(modelNumber);
      if (!designProfile) {
        // Keep the existing umbrella-style fallback for covered pergolas in
        // other parks that do not have one of the Qadisiyah APP_MODEL values.
        this.appendStandaloneGenericPergolaCanopyGeometry(
          localRings,
          modelData,
          height,
          positions,
          linePositions,
        );
        return;
      }

      const polygonArea = Math.abs(
        outer.reduce((sum, point, index) => {
          const next = outer[(index + 1) % outer.length];
          return sum + point.x * next.y - next.x * point.y;
        }, 0) / 2,
      );
      if (!Number.isFinite(polygonArea) || polygonArea < 1) {
        this.appendStandaloneGenericPergolaCanopyGeometry(
          localRings,
          modelData,
          height,
          positions,
          linePositions,
        );
        return;
      }

      // Use the longest GIS boundary segment as one structural-grid axis.
      // Every QAD footprint is drawn on the same 9.6 m modular grid, even when
      // the whole pergola is rotated in the site plan.
      let longestEdge = null;
      outer.forEach((point, index) => {
        const next = outer[(index + 1) % outer.length];
        const dx = Number(next.x) - Number(point.x);
        const dy = Number(next.y) - Number(point.y);
        const length = Math.hypot(dx, dy);
        if (!longestEdge || length > longestEdge.length) {
          longestEdge = { dx, dy, length };
        }
      });

      if (!longestEdge || longestEdge.length < 0.5) {
        this.appendStandaloneGenericPergolaCanopyGeometry(
          localRings,
          modelData,
          height,
          positions,
          linePositions,
        );
        return;
      }

      const axisU = {
        x: longestEdge.dx / longestEdge.length,
        y: longestEdge.dy / longestEdge.length,
      };
      const axisV = { x: -axisU.y, y: axisU.x };
      const projectPoint = (point) => ({
        x: Number(point.x) * axisU.x + Number(point.y) * axisU.y,
        y: Number(point.x) * axisV.x + Number(point.y) * axisV.y,
      });
      const projectedOuter = outer.map(projectPoint);
      const projectedHoles = localRings
        .slice(1)
        .filter((ring) => Array.isArray(ring) && ring.length >= 3)
        .map((ring) => ring.map(projectPoint));

      const uValues = projectedOuter.map((point) => point.x);
      const vValues = projectedOuter.map((point) => point.y);
      const minU = Math.min(...uValues);
      const maxU = Math.max(...uValues);
      const minV = Math.min(...vValues);
      const maxV = Math.max(...vValues);

      const moduleSize = Math.sqrt(
        polygonArea / Math.max(1, Number(designProfile.moduleCount) || 1),
      );
      const columns = Math.max(1, Math.round((maxU - minU) / moduleSize));
      const rows = Math.max(1, Math.round((maxV - minV) / moduleSize));

      const gridToLocal = (gridX, gridY) => {
        const projectedU = minU + gridX * moduleSize;
        const projectedV = minV + gridY * moduleSize;
        return new THREE.Vector2(
          axisU.x * projectedU + axisV.x * projectedV,
          axisU.y * projectedU + axisV.y * projectedV,
        );
      };

      const occupiedCells = [];
      for (let row = 0; row < rows; row += 1) {
        for (let column = 0; column < columns; column += 1) {
          const center = {
            x: minU + (column + 0.5) * moduleSize,
            y: minV + (row + 0.5) * moduleSize,
          };
          if (
            !this.isStandalonePointInsidePolygonRing(center, projectedOuter)
          ) {
            continue;
          }
          if (
            projectedHoles.some((hole) =>
              this.isStandalonePointInsidePolygonRing(center, hole),
            )
          ) {
            continue;
          }
          occupiedCells.push({ column, row });
        }
      }

      // The detail drawings and GIS polygons agree exactly on module count.
      // If a future dataset does not, preserve the old generic canopy instead
      // of producing a malformed detailed model.
      if (occupiedCells.length !== designProfile.moduleCount) {
        this.appendStandaloneGenericPergolaCanopyGeometry(
          localRings,
          modelData,
          height,
          positions,
          linePositions,
        );
        return;
      }

      const supportClearance = Math.max(4.4, Number(height) || 0);
      const canopyRise = Math.max(1.2, Math.min(1.8, supportClearance * 0.22));
      const usedNodeKeys = new Set();
      occupiedCells.forEach(({ column, row }) => {
        usedNodeKeys.add(`${column}|${row}`);
        usedNodeKeys.add(`${column + 1}|${row}`);
        usedNodeKeys.add(`${column + 1}|${row + 1}`);
        usedNodeKeys.add(`${column}|${row + 1}`);
      });

      // Use one level datum for the complete canopy so the high/low folded
      // silhouette stays faithful to the architectural elevations even where
      // the landscape terrain below it slopes.
      let highestTerrainY = -Infinity;
      usedNodeKeys.forEach((key) => {
        const [gridX, gridY] = key.split("|").map(Number);
        const point = gridToLocal(gridX, gridY);
        const terrainY = this.getStandaloneTerrainHeightAtPoint(
          { x: point.x, z: point.y },
          modelData,
        );
        if (Number.isFinite(Number(terrainY))) {
          highestTerrainY = Math.max(highestTerrainY, Number(terrainY));
        }
      });
      if (!Number.isFinite(highestTerrainY)) highestTerrainY = 0;

      const lowRoofY =
        highestTerrainY +
        THREE_STRUCTURE_SURFACE_CLEARANCE_METERS +
        supportClearance;
      const getNodeRoofY = (gridX, gridY) => {
        const highNode =
          (gridX + gridY + Number(designProfile.phase || 0)) % 2 === 0;
        return lowRoofY + (highNode ? canopyRise : 0);
      };
      const getNode = (gridX, gridY) => {
        const point = gridToLocal(gridX, gridY);
        return {
          x: point.x,
          z: point.y,
          y: getNodeRoofY(gridX, gridY),
          gridX,
          gridY,
        };
      };
      const pushTriangle = (a, b, c) => {
        positions.push(a.x, a.y, a.z, b.x, b.y, b.z, c.x, c.y, c.z);
      };
      const pushLine = (a, b, lift = 0.018) => {
        linePositions.push(a.x, a.y + lift, a.z, b.x, b.y + lift, b.z);
      };

      const uniqueRoofEdges = new Map();
      const addRoofEdge = (nodeA, nodeB) => {
        const keyA = `${nodeA.gridX}|${nodeA.gridY}`;
        const keyB = `${nodeB.gridX}|${nodeB.gridY}`;
        const key = keyA < keyB ? `${keyA}>${keyB}` : `${keyB}>${keyA}`;
        if (!uniqueRoofEdges.has(key)) uniqueRoofEdges.set(key, [nodeA, nodeB]);
      };

      occupiedCells.forEach(({ column, row }) => {
        const southWest = getNode(column, row);
        const southEast = getNode(column + 1, row);
        const northEast = getNode(column + 1, row + 1);
        const northWest = getNode(column, row + 1);

        addRoofEdge(southWest, southEast);
        addRoofEdge(southEast, northEast);
        addRoofEdge(northEast, northWest);
        addRoofEdge(northWest, southWest);

        const southWestIsHigh =
          (column + row + Number(designProfile.phase || 0)) % 2 === 0;
        const useHighRidge =
          (column + row + Number(designProfile.foldPhase || 0)) % 2 === 0;

        // Each 9.6 m bay is the same folded tensile/faceted roof module shown
        // in the detail sheets. Opposite grid nodes alternate high and low;
        // choosing the high-high or low-low diagonal gives the visible ridge /
        // valley folds seen in the axonometric and elevation drawings.
        let diagonalStart;
        let diagonalEnd;
        if (useHighRidge) {
          diagonalStart = southWestIsHigh ? southWest : southEast;
          diagonalEnd = southWestIsHigh ? northEast : northWest;
        } else {
          diagonalStart = southWestIsHigh ? southEast : southWest;
          diagonalEnd = southWestIsHigh ? northWest : northEast;
        }

        const diagonalIsSouthWestNorthEast =
          diagonalStart === southWest || diagonalStart === northEast;
        if (diagonalIsSouthWestNorthEast) {
          pushTriangle(southWest, southEast, northEast);
          pushTriangle(southWest, northEast, northWest);
        } else {
          pushTriangle(southWest, southEast, northWest);
          pushTriangle(southEast, northEast, northWest);
        }
        pushLine(diagonalStart, diagonalEnd, 0.026);
      });

      uniqueRoofEdges.forEach(([nodeA, nodeB]) => pushLine(nodeA, nodeB));

      // The detail drawings use a post at the structural grid nodes rather than
      // only four generic perimeter poles. Keep the posts thin so the existing
      // Qadisiyah visual style remains light and the canopy stays dominant.
      usedNodeKeys.forEach((key) => {
        const [gridX, gridY] = key.split("|").map(Number);
        const node = getNode(gridX, gridY);
        const terrainY =
          this.getStandaloneTerrainHeightAtPoint(
            { x: node.x, z: node.z },
            modelData,
          ) + THREE_STRUCTURE_SURFACE_CLEARANCE_METERS;
        const halfWidth = 0.065;

        positions.push(
          node.x - halfWidth,
          node.y,
          node.z,
          node.x - halfWidth,
          terrainY,
          node.z,
          node.x + halfWidth,
          terrainY,
          node.z,

          node.x - halfWidth,
          node.y,
          node.z,
          node.x + halfWidth,
          terrainY,
          node.z,
          node.x + halfWidth,
          node.y,
          node.z,

          node.x,
          node.y,
          node.z - halfWidth,
          node.x,
          terrainY,
          node.z - halfWidth,
          node.x,
          terrainY,
          node.z + halfWidth,

          node.x,
          node.y,
          node.z - halfWidth,
          node.x,
          terrainY,
          node.z + halfWidth,
          node.x,
          node.y,
          node.z + halfWidth,
        );

        linePositions.push(
          node.x,
          terrainY,
          node.z,
          node.x,
          node.y + 0.018,
          node.z,
        );
      });
    },

    getStandaloneCaseInsensitiveProperty(properties = {}, keys = []) {
      const lookup = new Map(
        Object.entries(properties || {}).map(([key, value]) => [
          String(key).toLowerCase(),
          value,
        ]),
      );

      for (const key of keys) {
        const value = lookup.get(String(key).toLowerCase());
        if (value == null) continue;
        if (typeof value === "string") {
          const trimmed = value.trim();
          if (!trimmed || /^<?null>?$/i.test(trimmed)) continue;
          return trimmed;
        }
        return value;
      }

      return "";
    },

    getStandaloneAppModel(properties = {}) {
      return String(
        this.getStandaloneCaseInsensitiveProperty(properties, [
          "APP_MODEL",
          "app_model",
          "APP_3D_MODEL",
          "app_3d_model",
          "App 3D Model",
        ]) || "",
      )
        .trim()
        .toLowerCase()
        .replace(/[\s-]+/g, "_");
    },

    getStandaloneStructureDescriptor(featureType = "", properties = {}) {
      return [
        featureType,
        properties.APP_MODEL,
        properties.app_model,
        properties.APP_3D_MODEL,
        properties.app_3d_model,
        properties["App 3D Model"],
        properties.APP_DISPLAY_LABEL,
        properties.app_display_label,
        properties["App Display Label"],
        properties.APP_LABEL,
        properties.app_label,
        properties.LABEL,
        properties.LABEL_EN,
        properties.DISPLAY_LABEL,
        properties.FACILITY_NAME,
        properties.FACILITY_TYPE,
        properties.ASSET_NAME,
        properties.ASSET_TYPE,
        properties.CL_LAND_COVER_DETAILED,
        properties.CL_LANDUSE_DETAILED,
        properties.QC_REMARKS,
        properties.DC_REMARK,
        properties.Remarks,
        properties.Type,
        properties.type,
        properties.name,
        properties.name_en,
        properties.sourceLayer,
        properties.source_layer,
      ]
        .filter((value) => value != null && String(value).trim())
        .join(" ")
        .toLowerCase()
        .replace(/[_-]+/g, " ")
        .replace(/\s+/g, " ")
        .trim();
    },

    getStandaloneArchitecturalBuildingStyle(featureType = "", properties = {}) {
      const appModel = this.getStandaloneAppModel(properties);
      if (!appModel) return null;

      // These three APP_MODEL values are authored in ArcGIS specifically for
      // Qadisiyah 3. They are kept separate from the generic/modular building
      // system so the architecture sheets can drive the visible 3D form.
      const styles = {
        maintenance_building: {
          key: "maintenance-building",
          label: "Maintenance Building",
          kind: "maintenance",
          // O&M sheet: coloured plaster concrete / polymer-concrete facade.
          facadeColor: 0xd8ca8f,
          wallHeight: 3.62,
          roofThickness: 0.18,
          parapetHeight: 0.2,
        },
        maintenance_and_water_tanks: {
          key: "maintenance-and-water-tanks",
          label: "Maintenance and Water Tanks",
          kind: "maintenance",
          // Al Monasya detail report: the visible maintenance ground floor is
          // 3.45 m finished-floor-to-ceiling; the water tanks/pump spaces are
          // underground and therefore are not extruded above the park surface.
          facadeColor: 0xd8ca8f,
          wallHeight: 3.45,
          roofThickness: 0.18,
          parapetHeight: 0.2,
        },
        electrical_building: {
          key: "electrical-building",
          label: "Electrical Building",
          kind: "electrical",
          // SEC sheet: coloured plaster concrete, approximately 3.78 m to the
          // facade top plus the thin roof/parapet band.
          facadeColor: 0xd8ca8f,
          wallHeight: 3.78,
          roofThickness: 0.18,
          parapetHeight: 0.22,
        },
        water_tanks_building: {
          key: "water-tanks-building",
          label: "Water Tanks Building",
          kind: "water-tanks",
          // The tanks/pump rooms are underground. Only the ground-level HVAC /
          // stair access volume represented by the GIS polygon is rendered.
          facadeColor: 0xd8ca8f,
          wallHeight: 3.18,
          roofThickness: 0.18,
          parapetHeight: 0.2,
        },
      };

      return styles[appModel] || null;
    },

    createStandaloneArchitecturalBuildingGroup(
      localRings,
      modelData,
      style,
      properties = {},
    ) {
      if (!style || !Array.isArray(localRings) || !localRings.length) {
        return null;
      }

      const outer = localRings[0];
      const frame = this.getStandaloneModularBuildingFrame(outer);
      if (!frame) return null;

      const group = new THREE.Group();
      group.name = `architectural-building-${style.key}`;
      group.userData.architecturalBuilding = true;
      group.userData.detailedModelKey = style.key;
      group.userData.detailedModelLabel = style.label;

      // Architectural facility buildings must sit on the ground, not on the
      // highest terrain sample inside their footprint. Using the generic
      // getStandalonePolygonSupportHeight() here lifted the entire flat building
      // to the footprint's highest point, which made the downhill side visibly
      // float above paths/terrain. Sample the footprint and anchor these
      // buildings to the lowest local support elevation instead.
      const architecturalSupportSamples = [];
      outer.forEach((point) => {
        const terrainY = this.getStandaloneTerrainHeightAtPoint(
          { x: Number(point.x), z: Number(point.y) },
          modelData,
        );
        if (Number.isFinite(Number(terrainY))) {
          architecturalSupportSamples.push(Number(terrainY));
        }
      });
      const centerTerrainY = this.getStandaloneTerrainHeightAtPoint(
        { x: frame.center.x, z: frame.center.y },
        modelData,
      );
      if (Number.isFinite(Number(centerTerrainY))) {
        architecturalSupportSamples.push(Number(centerTerrainY));
      }
      const supportY = architecturalSupportSamples.length
        ? Math.min(...architecturalSupportSamples) +
          THREE_STRUCTURE_SURFACE_CLEARANCE_METERS
        : THREE_STRUCTURE_SURFACE_CLEARANCE_METERS;
      group.position.set(frame.center.x, supportY, frame.center.y);
      group.rotation.y = frame.rotationY;

      // Fit the design inside the exact ArcGIS polygon. This intentionally
      // scales the reference architecture to the surveyed footprint because
      // the two Electrical Building polygons are not identical in area.
      const length = Math.max(2.5, frame.length * 0.975);
      const width = Math.max(2.5, frame.width * 0.975);
      const halfLength = length / 2;
      const halfWidth = width / 2;
      const plinthHeight = 0.14;
      const wallHeight = Math.max(2.8, Number(style.wallHeight) || 3.5);
      const roofThickness = Math.max(0.12, Number(style.roofThickness) || 0.18);
      const parapetHeight = Math.max(0.12, Number(style.parapetHeight) || 0.2);

      const facadeColor = new THREE.Color(style.facadeColor || 0xd8ca8f);
      const roofColor = facadeColor.clone().offsetHSL(0, -0.035, 0.045);
      const trimColor = facadeColor.clone().multiplyScalar(0.62);

      const facadeMaterial = new THREE.MeshStandardMaterial({
        color: facadeColor,
        emissive: facadeColor.clone().multiplyScalar(0.04),
        emissiveIntensity: 0.08,
        roughness: 0.92,
        metalness: 0.01,
      });
      const roofMaterial = new THREE.MeshStandardMaterial({
        color: roofColor,
        emissive: roofColor.clone().multiplyScalar(0.03),
        emissiveIntensity: 0.06,
        roughness: 0.95,
        metalness: 0.01,
      });
      const plinthMaterial = new THREE.MeshStandardMaterial({
        color: 0x777b79,
        roughness: 0.98,
        metalness: 0,
      });
      const doorMaterial = new THREE.MeshStandardMaterial({
        color: 0xc8c6b8,
        roughness: 0.78,
        metalness: 0.05,
        side: THREE.DoubleSide,
      });
      const serviceDoorMaterial = new THREE.MeshStandardMaterial({
        color: 0xa7a99f,
        roughness: 0.82,
        metalness: 0.07,
        side: THREE.DoubleSide,
      });
      const glassMaterial = new THREE.MeshStandardMaterial({
        color: 0x58717a,
        emissive: 0x101c20,
        emissiveIntensity: 0.12,
        roughness: 0.3,
        metalness: 0.08,
        transparent: true,
        opacity: 0.9,
        side: THREE.DoubleSide,
      });
      const ventMaterial = new THREE.MeshStandardMaterial({
        color: 0x8e9696,
        roughness: 0.68,
        metalness: 0.32,
        side: THREE.DoubleSide,
      });
      const lineMaterial = new THREE.LineBasicMaterial({
        color: trimColor,
        transparent: true,
        opacity: 0.48,
        depthWrite: false,
      });

      const plinth = new THREE.Mesh(
        new THREE.BoxGeometry(length * 1.01, plinthHeight, width * 1.01),
        plinthMaterial,
      );
      plinth.name = `${style.key}-plinth`;
      plinth.position.y = plinthHeight / 2;
      plinth.renderOrder = 19;
      group.add(plinth);

      const walls = new THREE.Mesh(
        new THREE.BoxGeometry(length, wallHeight, width),
        facadeMaterial,
      );
      walls.name = `${style.key}-facades`;
      walls.position.y = plinthHeight + wallHeight / 2;
      walls.renderOrder = 20;
      walls.frustumCulled = false;
      group.add(walls);

      // All three reference buildings use a flat non-accessible roof with a
      // thin perimeter/parapet band rather than the sloped modular roof.
      const roofY = plinthHeight + wallHeight;
      const roof = new THREE.Mesh(
        new THREE.BoxGeometry(length * 1.012, roofThickness, width * 1.012),
        roofMaterial,
      );
      roof.name = `${style.key}-flat-roof`;
      roof.position.y = roofY + roofThickness / 2;
      roof.renderOrder = 21;
      group.add(roof);

      const parapetY = roofY + roofThickness + parapetHeight / 2;
      const parapetThickness = Math.max(
        0.09,
        Math.min(0.16, Math.min(length, width) * 0.026),
      );
      [
        {
          geometry: new THREE.BoxGeometry(
            length * 1.02,
            parapetHeight,
            parapetThickness,
          ),
          x: 0,
          z: -halfWidth,
        },
        {
          geometry: new THREE.BoxGeometry(
            length * 1.02,
            parapetHeight,
            parapetThickness,
          ),
          x: 0,
          z: halfWidth,
        },
        {
          geometry: new THREE.BoxGeometry(
            parapetThickness,
            parapetHeight,
            width * 1.02,
          ),
          x: -halfLength,
          z: 0,
        },
        {
          geometry: new THREE.BoxGeometry(
            parapetThickness,
            parapetHeight,
            width * 1.02,
          ),
          x: halfLength,
          z: 0,
        },
      ].forEach((part, index) => {
        const parapet = new THREE.Mesh(part.geometry, roofMaterial);
        parapet.name = `${style.key}-parapet-${index + 1}`;
        parapet.position.set(part.x, parapetY, part.z);
        parapet.renderOrder = 22;
        group.add(parapet);
      });

      const addOpening = (
        side,
        centerRatio,
        widthRatio,
        height,
        bottom,
        material,
        name,
      ) => {
        const onLongFacade = side === "front" || side === "back";
        const span = onLongFacade ? length : width;
        const openingWidth = Math.max(
          0.48,
          Math.min(span * 0.52, span * widthRatio),
        );
        const openingHeight = Math.max(
          0.42,
          Math.min(Number(height) || 2, wallHeight - bottom - 0.1),
        );
        const center = THREE.MathUtils.lerp(
          -span / 2 + openingWidth / 2,
          span / 2 - openingWidth / 2,
          THREE.MathUtils.clamp(centerRatio, 0, 1),
        );
        const opening = new THREE.Mesh(
          new THREE.PlaneGeometry(openingWidth, openingHeight),
          material,
        );
        opening.name = `${style.key}-${name}`;
        opening.position.y = plinthHeight + bottom + openingHeight / 2;

        if (side === "front") {
          opening.position.x = center;
          opening.position.z = -halfWidth - 0.012;
          opening.rotation.y = Math.PI;
        } else if (side === "back") {
          opening.position.x = center;
          opening.position.z = halfWidth + 0.012;
        } else if (side === "left") {
          opening.position.x = -halfLength - 0.012;
          opening.position.z = center;
          opening.rotation.y = -Math.PI / 2;
        } else {
          opening.position.x = halfLength + 0.012;
          opening.position.z = center;
          opening.rotation.y = Math.PI / 2;
        }

        opening.renderOrder = 24;
        group.add(opening);
      };

      const addDoorSeam = (side, centerRatio, widthRatio, height, name) => {
        const onLongFacade = side === "front" || side === "back";
        const span = onLongFacade ? length : width;
        const center = THREE.MathUtils.lerp(
          -span / 2,
          span / 2,
          THREE.MathUtils.clamp(centerRatio, 0, 1),
        );
        const bottom = plinthHeight + 0.1;
        const top = Math.min(plinthHeight + wallHeight - 0.1, bottom + height);
        const points =
          side === "front" || side === "back"
            ? [
                new THREE.Vector3(
                  center,
                  bottom,
                  side === "front" ? -halfWidth - 0.018 : halfWidth + 0.018,
                ),
                new THREE.Vector3(
                  center,
                  top,
                  side === "front" ? -halfWidth - 0.018 : halfWidth + 0.018,
                ),
              ]
            : [
                new THREE.Vector3(
                  side === "left" ? -halfLength - 0.018 : halfLength + 0.018,
                  bottom,
                  center,
                ),
                new THREE.Vector3(
                  side === "left" ? -halfLength - 0.018 : halfLength + 0.018,
                  top,
                  center,
                ),
              ];
        const seam = new THREE.Line(
          new THREE.BufferGeometry().setFromPoints(points),
          lineMaterial,
        );
        seam.name = `${style.key}-${name}-seam`;
        seam.renderOrder = 25;
        group.add(seam);
      };

      if (style.kind === "electrical") {
        // SEC Building sheet: centred double access opening on the front
        // elevation, high ventilation grille on the opposite elevation.
        addOpening(
          "right",
          0.5,
          0.46,
          2.38,
          0.08,
          doorMaterial,
          "front-double-door",
        );
        addDoorSeam("right", 0.5, 0.46, 2.38, "front-double-door");
        addOpening(
          "left",
          0.5,
          0.46,
          0.9,
          Math.max(2.25, wallHeight - 1.18),
          ventMaterial,
          "rear-vent",
        );
      } else if (style.kind === "water-tanks") {
        // The visible ground-level HVAC/stair enclosure is plain on the long
        // elevations, with access doors on the short elevations.
        ["left", "right"].forEach((side) => {
          addOpening(
            side,
            0.5,
            0.5,
            2.32,
            0.08,
            doorMaterial,
            `${side}-double-door`,
          );
          addDoorSeam(side, 0.5, 0.5, 2.32, `${side}-double-door`);
        });
      } else if (style.kind === "maintenance") {
        // O&M south elevation: three broad storage/service openings followed by
        // the administration door/window rhythm shown on the architecture sheet.
        [
          [0.09, 0.145],
          [0.255, 0.145],
          [0.42, 0.145],
        ].forEach(([centerRatio, widthRatio], index) => {
          addOpening(
            "front",
            centerRatio,
            widthRatio,
            2.62,
            0.1,
            serviceDoorMaterial,
            `storage-door-${index + 1}`,
          );
        });
        addOpening(
          "front",
          0.565,
          0.052,
          2.24,
          0.08,
          doorMaterial,
          "office-door",
        );
        [0.67, 0.765, 0.865, 0.94].forEach((centerRatio, index) => {
          addOpening(
            "front",
            centerRatio,
            0.052,
            1.3,
            0.92,
            glassMaterial,
            `south-window-${index + 1}`,
          );
        });

        // O&M north elevation: sparse service window and door arrangement.
        addOpening(
          "back",
          0.08,
          0.052,
          1.3,
          0.92,
          glassMaterial,
          "north-window",
        );
        addOpening(
          "back",
          0.285,
          0.09,
          2.34,
          0.08,
          doorMaterial,
          "north-door-a",
        );
        addDoorSeam("back", 0.285, 0.09, 2.34, "north-door-a");
        addOpening(
          "back",
          0.635,
          0.052,
          2.28,
          0.08,
          doorMaterial,
          "north-door-b",
        );
        addOpening(
          "back",
          0.805,
          0.09,
          2.34,
          0.08,
          doorMaterial,
          "north-door-c",
        );
        addDoorSeam("back", 0.805, 0.09, 2.34, "north-door-c");
      }

      // Fine roof/base lines keep the low, flat architecture readable at the
      // oblique park camera without turning it into a bright wireframe object.
      const outlinePoints = [
        new THREE.Vector3(-halfLength, plinthHeight + 0.02, -halfWidth - 0.02),
        new THREE.Vector3(halfLength, plinthHeight + 0.02, -halfWidth - 0.02),
        new THREE.Vector3(halfLength, plinthHeight + 0.02, halfWidth + 0.02),
        new THREE.Vector3(-halfLength, plinthHeight + 0.02, halfWidth + 0.02),
      ];
      const baseLoop = new THREE.LineLoop(
        new THREE.BufferGeometry().setFromPoints(outlinePoints),
        lineMaterial,
      );
      baseLoop.name = `${style.key}-base-outline`;
      baseLoop.renderOrder = 23;
      group.add(baseLoop);

      const topOutline = new THREE.LineLoop(
        new THREE.BufferGeometry().setFromPoints(
          outlinePoints.map(
            (point) =>
              new THREE.Vector3(
                point.x,
                roofY + roofThickness + parapetHeight + 0.012,
                point.z,
              ),
          ),
        ),
        lineMaterial,
      );
      topOutline.name = `${style.key}-roof-outline`;
      topOutline.renderOrder = 23;
      group.add(topOutline);

      return group;
    },

    getStandaloneDetailedModularBuildingStyle(
      featureType = "",
      properties = {},
    ) {
      const descriptor = this.getStandaloneStructureDescriptor(
        featureType,
        properties,
      );
      if (!descriptor) return null;

      // These colours follow the Stage 3 Architecture Modular System sheets.
      // They deliberately bypass the normal purple STRUCTURES palette so the
      // park model communicates the specified facility type at a glance.
      const styles = {
        cyclingCenter: {
          key: "cycling-center-46",
          label: "Cycling Center",
          modules: 2,
          facadeColor: 0x2f4538, // RAL 6025 - Fern green
          eaveHeight: 3.55,
          ridgeRise: 1.08,
          crossRise: 0.28,
          openingMode: "cycling",
        },
        administrationOffice: {
          key: "administration-office-visitor-center-46",
          label: "Administration Office / Visitor Center",
          modules: 2,
          facadeColor: 0xb89c50, // RAL 1024 - Ochre yellow
          eaveHeight: 3.55,
          ridgeRise: 1.02,
          crossRise: 0.24,
          openingMode: "office",
        },
        visitorCenter: {
          key: "visitor-center-23",
          label: "Visitor Center",
          modules: 1,
          facadeColor: 0x6093ac, // RAL 5024 - Pastel blue
          eaveHeight: 3.5,
          ridgeRise: 0.92,
          crossRise: 0.2,
          openingMode: "visitor",
        },
        largeKiosk: {
          key: "large-kiosk-46",
          label: "Large Kiosk",
          modules: 2,
          facadeColor: 0xc63927, // RAL 2001 - Red orange
          eaveHeight: 3.5,
          ridgeRise: 1.0,
          crossRise: 0.24,
          openingMode: "large-kiosk",
        },
        smallKiosk: {
          key: "small-kiosk-23",
          label: "Small Kiosk",
          modules: 1,
          facadeColor: 0xc63927, // RAL 2001 - Red orange
          eaveHeight: 3.45,
          ridgeRise: 0.9,
          crossRise: 0.18,
          openingMode: "small-kiosk",
        },
        doubleToilet: {
          key: "double-toilet-23",
          label: "Double Toilet 23m²",
          modules: 1,
          facadeColor: 0x1a5784, // RAL 5009 - Azure blue
          eaveHeight: 3.45,
          ridgeRise: 0.9,
          crossRise: 0.18,
          openingMode: "toilet",
        },
        simpleToilets: {
          key: "two-simple-toilets-46",
          label: "2 Simple Toilets 46m²",
          modules: 2,
          facadeColor: 0x1a5784, // RAL 5009 - Azure blue
          eaveHeight: 3.5,
          ridgeRise: 1.0,
          crossRise: 0.22,
          openingMode: "toilets",
        },
        simpleToiletsAblution: {
          key: "two-simple-toilets-ablution-46",
          label: "2 Simple Toilets / Ablution 46m²",
          modules: 2,
          facadeColor: 0x1a5784, // RAL 5009 - Azure blue
          eaveHeight: 3.5,
          ridgeRise: 1.0,
          crossRise: 0.22,
          openingMode: "toilets-ablution",
        },
        prayerRoom: {
          key: "prayer-room-46",
          label: "Prayer Room",
          modules: 2,
          facadeColor: 0x428c78, // RAL 6033 - Mint turquoise
          eaveHeight: 3.6,
          ridgeRise: 1.08,
          crossRise: 0.24,
          openingMode: "prayer",
        },
        electricalRoom: {
          key: "electrical-room-46",
          label: "Electrical Room",
          modules: 2,
          // Al Monasya modular-system Electrical room 46 m² sheet.
          // Ochre/yellow polymer-concrete facade with the same asymmetric
          // modular roof language used by the other detailed facilities.
          facadeColor: 0xd69a24,
          eaveHeight: 3.55,
          ridgeRise: 1.02,
          crossRise: 0.22,
          openingMode: "electrical-room",
        },
      };

      if (
        /\bcycling\s+(?:center|centre)\b|\bcycle\s+(?:center|centre)\b/.test(
          descriptor,
        )
      ) {
        return styles.cyclingCenter;
      }
      if (
        /\badministration\s+(?:office|building)\b/.test(descriptor) ||
        (/\badministration\b/.test(descriptor) &&
          /\bvisitor\s+center\b/.test(descriptor))
      ) {
        return styles.administrationOffice;
      }
      if (/\bvisitor\s+(?:center|centre)\b/.test(descriptor)) {
        return styles.visitorCenter;
      }
      if (/\blarge\s+kiosk\b/.test(descriptor)) return styles.largeKiosk;
      if (/\bsmall\s+kiosk\b/.test(descriptor)) return styles.smallKiosk;

      // Qadisiyah bathroom GIS naming does not match the Architecture-sheet
      // naming: the ~23 m² footprints are stored as SIMPLE TOILET, while the
      // ~46 m² footprints are stored as DOUBLE TOILET. Select the detailed
      // bathroom architecture primarily from the actual GIS footprint area so
      // every bathroom gets the correct RAL 5009 modular design.
      const bathroomArea = Number(
        this.getStandaloneCaseInsensitiveProperty(properties, [
          "SHAPE_Area",
          "SHAPE_AREA",
          "shape_area",
          "Shape_Area",
          "area_m2",
          "AREA_M2",
          "Area",
          "AREA",
        ]),
      );
      const isBathroomDescriptor =
        /\b(?:simple|double|two|2)?\s*toilet(?:s)?\b|\bbathroom(?:s)?\b|\bablution\b/.test(
          descriptor,
        );

      if (isBathroomDescriptor && Number.isFinite(bathroomArea)) {
        // Architecture sheet: Double Toilet = approximately 23 m².
        if (bathroomArea < 34) {
          return styles.doubleToilet;
        }

        // Both remaining detailed bathroom designs are approximately 46 m².
        // Use the Ablution version whenever the GIS/app attributes explicitly
        // identify ablution; otherwise use the 2 Simple Toilets design.
        if (/\bablution\b/.test(descriptor)) {
          return styles.simpleToiletsAblution;
        }
        return styles.simpleToilets;
      }

      // Fallbacks for future projects that already use the Architecture naming.
      if (
        /\b(?:2|two)\s+simple\s+toilet(?:s)?\b/.test(descriptor) &&
        /\bablution\b/.test(descriptor)
      ) {
        return styles.simpleToiletsAblution;
      }
      if (/\b(?:2|two)\s+simple\s+toilet(?:s)?\b/.test(descriptor)) {
        return styles.simpleToilets;
      }
      if (/\bdouble\s+toilet(?:s)?\b/.test(descriptor)) {
        return styles.doubleToilet;
      }
      if (
        /\bsimple\s+toilet(?:s)?\b/.test(descriptor) &&
        /\bablution\b/.test(descriptor)
      ) {
        return styles.simpleToiletsAblution;
      }
      if (/\bsimple\s+toilet(?:s)?\b|\bbathroom(?:s)?\b/.test(descriptor)) {
        return styles.doubleToilet;
      }

      if (/\bprayer\s+room\b|\bprayer\s+hall\b/.test(descriptor)) {
        return styles.prayerRoom;
      }
      if (/\belectrical\s+room\b/.test(descriptor)) {
        return styles.electricalRoom;
      }

      return null;
    },

    getStandaloneModularBuildingFrame(outer = []) {
      if (!Array.isArray(outer) || outer.length < 3) return null;
      const centroid = this.getStandalonePolygonCentroid(outer);
      if (!centroid) return null;

      let xx = 0;
      let xy = 0;
      let yy = 0;
      outer.forEach((point) => {
        const dx = Number(point.x) - centroid.x;
        const dy = Number(point.y) - centroid.y;
        xx += dx * dx;
        xy += dx * dy;
        yy += dy * dy;
      });

      const angle = 0.5 * Math.atan2(2 * xy, xx - yy);
      let longAxis = new THREE.Vector2(
        Math.cos(angle),
        Math.sin(angle),
      ).normalize();
      let shortAxis = new THREE.Vector2(-longAxis.y, longAxis.x).normalize();

      const measure = (axisA, axisB) => {
        let minA = Infinity;
        let maxA = -Infinity;
        let minB = Infinity;
        let maxB = -Infinity;
        outer.forEach((point) => {
          const relative = new THREE.Vector2(
            Number(point.x) - centroid.x,
            Number(point.y) - centroid.y,
          );
          const a = relative.dot(axisA);
          const b = relative.dot(axisB);
          minA = Math.min(minA, a);
          maxA = Math.max(maxA, a);
          minB = Math.min(minB, b);
          maxB = Math.max(maxB, b);
        });
        return { minA, maxA, minB, maxB };
      };

      let bounds = measure(longAxis, shortAxis);
      if (bounds.maxA - bounds.minA < bounds.maxB - bounds.minB) {
        const previousLong = longAxis;
        longAxis = shortAxis;
        shortAxis = previousLong.clone().multiplyScalar(-1);
        bounds = measure(longAxis, shortAxis);
      }

      const centerA = (bounds.minA + bounds.maxA) / 2;
      const centerB = (bounds.minB + bounds.maxB) / 2;
      const center = centroid
        .clone()
        .add(longAxis.clone().multiplyScalar(centerA))
        .add(shortAxis.clone().multiplyScalar(centerB));

      return {
        center,
        longAxis,
        shortAxis,
        rotationY: -Math.atan2(longAxis.y, longAxis.x),
        length: Math.max(2.5, bounds.maxA - bounds.minA),
        width: Math.max(2.5, bounds.maxB - bounds.minB),
      };
    },

    createStandaloneDetailedModularBuildingGroup(
      localRings,
      modelData,
      style,
      properties = {},
    ) {
      if (!style || !Array.isArray(localRings) || !localRings.length) {
        return null;
      }
      const outer = localRings[0];
      const frame = this.getStandaloneModularBuildingFrame(outer);
      if (!frame) return null;

      const group = new THREE.Group();
      group.name = `detailed-modular-building-${style.key}`;
      group.userData.detailedModularBuilding = true;
      group.userData.detailedModelKey = style.key;
      group.userData.detailedModelLabel = style.label;

      const terrainSupportY =
        this.getStandalonePolygonSupportHeight(localRings, modelData) +
        THREE_STRUCTURE_SURFACE_CLEARANCE_METERS;
      group.position.set(frame.center.x, terrainSupportY, frame.center.y);
      group.rotation.y = frame.rotationY;

      // Keep the procedural module just inside the GIS footprint so odd ArcGIS
      // ring vertices never make the building protrude beyond the planned pad.
      const halfLength = Math.max(1.2, frame.length * 0.485);
      const halfWidth = Math.max(1.2, frame.width * 0.485);
      const eaveHeight = Math.max(2.8, Number(style.eaveHeight) || 3.5);
      const ridgeRise = Math.max(0.45, Number(style.ridgeRise) || 0.9);
      const crossRise = Math.max(0, Number(style.crossRise) || 0);
      const moduleCount = Number(style.modules) === 1 ? 1 : 2;

      const roofYAt = (x, z) => {
        const xNorm = THREE.MathUtils.clamp(x / halfLength, -1, 1);
        const zNorm = THREE.MathUtils.clamp(z / halfWidth, -1, 1);
        if (moduleCount === 1) {
          // Single 23 m² modules use the mono-pitch roof visible in the detail
          // elevations, with a very small secondary fall across the facade.
          return (
            eaveHeight +
            ridgeRise * ((xNorm + 1) / 2) +
            crossRise * ((zNorm + 1) / 2)
          );
        }

        // Two-module 46 m² buildings meet at a raised central seam. A slight
        // cross-fall reproduces the asymmetric elevations from the design set.
        const ridgeFactor = 1 - Math.abs(xNorm);
        return (
          eaveHeight + ridgeRise * ridgeFactor + crossRise * ((zNorm + 1) / 2)
        );
      };

      const facadeColor = new THREE.Color(style.facadeColor);
      const roofColor = facadeColor.clone().offsetHSL(0, -0.035, 0.055);
      const panelLineColor = facadeColor.clone().multiplyScalar(0.58);
      const facadeMaterial = new THREE.MeshStandardMaterial({
        color: facadeColor,
        emissive: facadeColor.clone().multiplyScalar(0.07),
        emissiveIntensity: 0.12,
        roughness: 0.88,
        metalness: 0.02,
        side: THREE.DoubleSide,
      });
      const roofMaterial = new THREE.MeshStandardMaterial({
        color: roofColor,
        emissive: facadeColor.clone().multiplyScalar(0.055),
        emissiveIntensity: 0.1,
        roughness: 0.9,
        metalness: 0.02,
        side: THREE.DoubleSide,
      });
      const panelLineMaterial = new THREE.LineBasicMaterial({
        color: panelLineColor,
        transparent: true,
        opacity: 0.5,
        depthWrite: false,
      });

      const wallPositions = [];
      const roofPositions = [];
      const detailLines = [];
      const pushQuad = (target, a, b, c, d) => {
        target.push(...a, ...b, ...c, ...a, ...c, ...d);
      };

      const frontZ = -halfWidth;
      const backZ = halfWidth;
      const xStops =
        moduleCount === 2
          ? [-halfLength, 0, halfLength]
          : [-halfLength, halfLength];

      // Long facades. Splitting at the module seam preserves the ridge shape.
      [frontZ, backZ].forEach((z) => {
        for (let index = 0; index < xStops.length - 1; index += 1) {
          const x1 = xStops[index];
          const x2 = xStops[index + 1];
          pushQuad(
            wallPositions,
            [x1, 0, z],
            [x2, 0, z],
            [x2, roofYAt(x2, z), z],
            [x1, roofYAt(x1, z), z],
          );
        }
      });

      // End facades follow the small cross-fall of the roof.
      [-halfLength, halfLength].forEach((x) => {
        pushQuad(
          wallPositions,
          [x, 0, frontZ],
          [x, 0, backZ],
          [x, roofYAt(x, backZ), backZ],
          [x, roofYAt(x, frontZ), frontZ],
        );
      });

      // Roof planes. Two-module facilities get a visible center seam/ridge.
      if (moduleCount === 2) {
        pushQuad(
          roofPositions,
          [-halfLength, roofYAt(-halfLength, frontZ), frontZ],
          [0, roofYAt(0, frontZ), frontZ],
          [0, roofYAt(0, backZ), backZ],
          [-halfLength, roofYAt(-halfLength, backZ), backZ],
        );
        pushQuad(
          roofPositions,
          [0, roofYAt(0, frontZ), frontZ],
          [halfLength, roofYAt(halfLength, frontZ), frontZ],
          [halfLength, roofYAt(halfLength, backZ), backZ],
          [0, roofYAt(0, backZ), backZ],
        );
        detailLines.push(
          0,
          roofYAt(0, frontZ) + 0.018,
          frontZ,
          0,
          roofYAt(0, backZ) + 0.018,
          backZ,
        );
      } else {
        pushQuad(
          roofPositions,
          [-halfLength, roofYAt(-halfLength, frontZ), frontZ],
          [halfLength, roofYAt(halfLength, frontZ), frontZ],
          [halfLength, roofYAt(halfLength, backZ), backZ],
          [-halfLength, roofYAt(-halfLength, backZ), backZ],
        );
      }

      const wallGeometry = new THREE.BufferGeometry();
      wallGeometry.setAttribute(
        "position",
        new THREE.Float32BufferAttribute(wallPositions, 3),
      );
      wallGeometry.computeVertexNormals();
      const walls = new THREE.Mesh(wallGeometry, facadeMaterial);
      walls.name = `${style.key}-facades`;
      walls.renderOrder = 20;
      walls.frustumCulled = false;
      group.add(walls);

      const roofGeometry = new THREE.BufferGeometry();
      roofGeometry.setAttribute(
        "position",
        new THREE.Float32BufferAttribute(roofPositions, 3),
      );
      roofGeometry.computeVertexNormals();
      const roof = new THREE.Mesh(roofGeometry, roofMaterial);
      roof.name = `${style.key}-sloped-roof`;
      roof.renderOrder = 21;
      roof.frustumCulled = false;
      group.add(roof);

      // Exterior polymer-concrete panel rhythm from the elevation sheets.
      const panelSpacing = 1.15;
      [frontZ - 0.012, backZ + 0.012].forEach((z, sideIndex) => {
        const panelCount = Math.max(
          2,
          Math.round((halfLength * 2) / panelSpacing),
        );
        for (let index = 0; index <= panelCount; index += 1) {
          const x = THREE.MathUtils.lerp(
            -halfLength,
            halfLength,
            index / panelCount,
          );
          detailLines.push(
            x,
            0.08,
            z,
            x,
            roofYAt(x, sideIndex === 0 ? frontZ : backZ) - 0.035,
            z,
          );
        }
        const bandY = Math.min(eaveHeight * 0.5, 1.72);
        detailLines.push(-halfLength, bandY, z, halfLength, bandY, z);
      });

      [-halfLength - 0.012, halfLength + 0.012].forEach((x) => {
        const panelCount = Math.max(
          2,
          Math.round((halfWidth * 2) / panelSpacing),
        );
        for (let index = 0; index <= panelCount; index += 1) {
          const z = THREE.MathUtils.lerp(
            -halfWidth,
            halfWidth,
            index / panelCount,
          );
          detailLines.push(
            x,
            0.08,
            z,
            x,
            roofYAt(x > 0 ? halfLength : -halfLength, z) - 0.035,
            z,
          );
        }
        const bandY = Math.min(eaveHeight * 0.5, 1.72);
        detailLines.push(x, bandY, -halfWidth, x, bandY, halfWidth);
      });

      // Roof perimeter outlines make the changing roof height legible from the
      // same oblique camera angle used by the 3D park view.
      const roofOutlineStops =
        moduleCount === 2 ? xStops : [-halfLength, halfLength];
      [frontZ, backZ].forEach((z) => {
        for (let index = 0; index < roofOutlineStops.length - 1; index += 1) {
          const x1 = roofOutlineStops[index];
          const x2 = roofOutlineStops[index + 1];
          detailLines.push(
            x1,
            roofYAt(x1, z) + 0.018,
            z,
            x2,
            roofYAt(x2, z) + 0.018,
            z,
          );
        }
      });
      [-halfLength, halfLength].forEach((x) => {
        detailLines.push(
          x,
          roofYAt(x, frontZ) + 0.018,
          frontZ,
          x,
          roofYAt(x, backZ) + 0.018,
          backZ,
        );
      });

      if (detailLines.length) {
        const lineGeometry = new THREE.BufferGeometry();
        lineGeometry.setAttribute(
          "position",
          new THREE.Float32BufferAttribute(detailLines, 3),
        );
        const lines = new THREE.LineSegments(lineGeometry, panelLineMaterial);
        lines.name = `${style.key}-facade-panel-lines`;
        lines.renderOrder = 23;
        lines.frustumCulled = false;
        group.add(lines);
      }

      // A thin concrete plinth makes the building read as a constructed module
      // rather than a coloured polygon growing directly out of the terrain.
      const plinthMaterial = new THREE.MeshStandardMaterial({
        color: 0x72797c,
        roughness: 0.98,
        metalness: 0,
      });
      const plinth = new THREE.Mesh(
        new THREE.BoxGeometry(halfLength * 2.02, 0.14, halfWidth * 2.02),
        plinthMaterial,
      );
      plinth.name = `${style.key}-concrete-plinth`;
      plinth.position.y = 0.07;
      plinth.renderOrder = 19;
      group.add(plinth);

      const glassMaterial = new THREE.MeshStandardMaterial({
        color: 0x315a69,
        emissive: 0x0b2631,
        emissiveIntensity: 0.22,
        roughness: 0.28,
        metalness: 0.08,
        transparent: true,
        opacity: 0.88,
        side: THREE.DoubleSide,
        depthWrite: true,
      });
      const doorMaterial = new THREE.MeshStandardMaterial({
        // Keep bathroom and facility doors visibly distinct from the dark blue
        // RAL 5009 facade panels. The previous near-black material disappeared
        // into the bathroom facade at the normal park camera distance.
        color: 0xb7c0c3,
        emissive: 0x222b2f,
        emissiveIntensity: 0.08,
        roughness: 0.72,
        metalness: 0.08,
        side: THREE.DoubleSide,
      });
      const doorFrameMaterial = new THREE.MeshStandardMaterial({
        color: 0xe1e5e4,
        emissive: 0x303536,
        emissiveIntensity: 0.06,
        roughness: 0.68,
        metalness: 0.12,
        side: THREE.DoubleSide,
      });
      const doorSeamMaterial = new THREE.LineBasicMaterial({
        color: 0x4a5559,
        transparent: true,
        opacity: 0.92,
        depthWrite: false,
      });

      const addFrontOpening = (
        centerRatio,
        widthRatio,
        height,
        bottom,
        material,
        name,
      ) => {
        const openingWidth = Math.max(0.55, halfLength * 2 * widthRatio);
        const x = THREE.MathUtils.lerp(
          -halfLength + openingWidth / 2,
          halfLength - openingWidth / 2,
          THREE.MathUtils.clamp(centerRatio, 0, 1),
        );
        const maxWallHeight = roofYAt(x, frontZ) - 0.18;
        const openingHeight = Math.max(
          0.45,
          Math.min(height, maxWallHeight - bottom),
        );
        if (openingHeight <= 0.3) return;
        const opening = new THREE.Mesh(
          new THREE.PlaneGeometry(openingWidth, openingHeight),
          material,
        );
        opening.name = `${style.key}-${name}`;
        opening.position.set(x, bottom + openingHeight / 2, frontZ - 0.022);
        opening.rotation.y = Math.PI;
        opening.renderOrder = 24;
        group.add(opening);
      };

      // Bathroom doors are modeled as shallow 3D assemblies rather than flat
      // facade decals. This keeps them visible from oblique camera angles and
      // prevents them disappearing into the blue RAL 5009 wall surface.
      const addDetailedDoor = (
        side,
        centerRatio,
        widthRatio,
        height = 2.14,
        bottom = 0.08,
        name = "door",
        doubleLeaf = false,
      ) => {
        const onLongFacade = side === "front" || side === "back";
        const span = onLongFacade ? halfLength * 2 : halfWidth * 2;
        const openingWidth = Math.max(
          0.78,
          Math.min(span * 0.42, span * widthRatio),
        );
        const openingHeight = Math.max(
          1.75,
          Math.min(height, eaveHeight - bottom - 0.18),
        );
        if (openingHeight <= 0.8) return;

        const center = THREE.MathUtils.lerp(
          -span / 2 + openingWidth / 2,
          span / 2 - openingWidth / 2,
          THREE.MathUtils.clamp(centerRatio, 0, 1),
        );

        const frameThickness = Math.max(
          0.055,
          Math.min(0.11, openingWidth * 0.055),
        );
        const doorDepth = 0.09;

        const doorGeometry = new THREE.BoxGeometry(
          openingWidth,
          openingHeight,
          doorDepth,
        );
        const door = new THREE.Mesh(doorGeometry, doorMaterial);
        door.name = `${style.key}-${name}-leaf`;
        door.renderOrder = 26;

        if (side === "front") {
          door.position.set(
            center,
            bottom + openingHeight / 2,
            frontZ - doorDepth / 2 - 0.025,
          );
        } else if (side === "back") {
          door.position.set(
            center,
            bottom + openingHeight / 2,
            backZ + doorDepth / 2 + 0.025,
          );
        } else if (side === "left") {
          door.geometry.dispose();
          door.geometry = new THREE.BoxGeometry(
            doorDepth,
            openingHeight,
            openingWidth,
          );
          door.position.set(
            -halfLength - doorDepth / 2 - 0.025,
            bottom + openingHeight / 2,
            center,
          );
        } else {
          door.geometry.dispose();
          door.geometry = new THREE.BoxGeometry(
            doorDepth,
            openingHeight,
            openingWidth,
          );
          door.position.set(
            halfLength + doorDepth / 2 + 0.025,
            bottom + openingHeight / 2,
            center,
          );
        }
        group.add(door);

        const addFrameBox = (width, height, depth, x, y, z, suffix) => {
          const frame = new THREE.Mesh(
            new THREE.BoxGeometry(width, height, depth),
            doorFrameMaterial,
          );
          frame.name = `${style.key}-${name}-${suffix}`;
          frame.position.set(x, y, z);
          frame.renderOrder = 27;
          group.add(frame);
        };

        if (onLongFacade) {
          const z =
            side === "front"
              ? frontZ - doorDepth - 0.045
              : backZ + doorDepth + 0.045;
          addFrameBox(
            frameThickness,
            openingHeight + frameThickness * 2,
            0.045,
            center - openingWidth / 2 - frameThickness / 2,
            bottom + openingHeight / 2,
            z,
            "frame-left",
          );
          addFrameBox(
            frameThickness,
            openingHeight + frameThickness * 2,
            0.045,
            center + openingWidth / 2 + frameThickness / 2,
            bottom + openingHeight / 2,
            z,
            "frame-right",
          );
          addFrameBox(
            openingWidth + frameThickness * 2,
            frameThickness,
            0.045,
            center,
            bottom + openingHeight + frameThickness / 2,
            z,
            "frame-top",
          );
        } else {
          const x =
            side === "left"
              ? -halfLength - doorDepth - 0.045
              : halfLength + doorDepth + 0.045;
          addFrameBox(
            0.045,
            openingHeight + frameThickness * 2,
            frameThickness,
            x,
            bottom + openingHeight / 2,
            center - openingWidth / 2 - frameThickness / 2,
            "frame-left",
          );
          addFrameBox(
            0.045,
            openingHeight + frameThickness * 2,
            frameThickness,
            x,
            bottom + openingHeight / 2,
            center + openingWidth / 2 + frameThickness / 2,
            "frame-right",
          );
          addFrameBox(
            0.045,
            frameThickness,
            openingWidth + frameThickness * 2,
            x,
            bottom + openingHeight + frameThickness / 2,
            center,
            "frame-top",
          );
        }

        if (doubleLeaf) {
          const seamPoints = onLongFacade
            ? [
                new THREE.Vector3(
                  center,
                  bottom + 0.05,
                  side === "front"
                    ? frontZ - doorDepth - 0.075
                    : backZ + doorDepth + 0.075,
                ),
                new THREE.Vector3(
                  center,
                  bottom + openingHeight - 0.05,
                  side === "front"
                    ? frontZ - doorDepth - 0.075
                    : backZ + doorDepth + 0.075,
                ),
              ]
            : [
                new THREE.Vector3(
                  side === "left"
                    ? -halfLength - doorDepth - 0.075
                    : halfLength + doorDepth + 0.075,
                  bottom + 0.05,
                  center,
                ),
                new THREE.Vector3(
                  side === "left"
                    ? -halfLength - doorDepth - 0.075
                    : halfLength + doorDepth + 0.075,
                  bottom + openingHeight - 0.05,
                  center,
                ),
              ];
          const seam = new THREE.Line(
            new THREE.BufferGeometry().setFromPoints(seamPoints),
            doorSeamMaterial,
          );
          seam.name = `${style.key}-${name}-center-seam`;
          seam.renderOrder = 28;
          group.add(seam);
        }

        const handleMaterial = new THREE.MeshStandardMaterial({
          color: 0x525d61,
          roughness: 0.42,
          metalness: 0.5,
        });
        const handleGeometry = new THREE.SphereGeometry(0.055, 8, 6);
        const handleOffsets = doubleLeaf
          ? [-openingWidth * 0.07, openingWidth * 0.07]
          : [openingWidth * 0.27];

        handleOffsets.forEach((offset, index) => {
          const handle = new THREE.Mesh(handleGeometry, handleMaterial);
          handle.name = `${style.key}-${name}-handle-${index + 1}`;
          if (onLongFacade) {
            handle.position.set(
              center + offset,
              bottom + openingHeight * 0.52,
              side === "front"
                ? frontZ - doorDepth - 0.11
                : backZ + doorDepth + 0.11,
            );
          } else {
            handle.position.set(
              side === "left"
                ? -halfLength - doorDepth - 0.11
                : halfLength + doorDepth + 0.11,
              bottom + openingHeight * 0.52,
              center + offset,
            );
          }
          handle.renderOrder = 29;
          group.add(handle);
        });
      };

      switch (style.openingMode) {
        case "electrical-room":
          // Electrical room 46 m² detail: paired sliding/service doors on the
          // long elevations plus ventilation openings on the short elevations.
          addDetailedDoor("front", 0.28, 0.22, 2.2, 0.08, "front-door-a", true);
          addDetailedDoor("front", 0.72, 0.22, 2.2, 0.08, "front-door-b", true);
          addDetailedDoor("back", 0.28, 0.22, 2.2, 0.08, "back-door-a", true);
          addDetailedDoor("back", 0.72, 0.22, 2.2, 0.08, "back-door-b", true);
          break;
        case "large-kiosk":
          addFrontOpening(
            0.64,
            0.34,
            1.28,
            1.02,
            glassMaterial,
            "service-window",
          );
          addFrontOpening(0.17, 0.12, 2.12, 0.08, doorMaterial, "door");
          break;
        case "small-kiosk":
          addFrontOpening(
            0.62,
            0.4,
            1.22,
            1.02,
            glassMaterial,
            "service-window",
          );
          addFrontOpening(0.16, 0.16, 2.08, 0.08, doorMaterial, "door");
          break;
        case "toilet":
          // Double Toilet 23 m²: the architecture plan shows two exterior
          // entrances total, positioned on opposite long facades (E02/E04).
          // Do not duplicate a male/female pair on both sides.
          addDetailedDoor("front", 0.62, 0.19, 2.14, 0.08, "front-door");
          addDetailedDoor("back", 0.62, 0.19, 2.14, 0.08, "back-door");
          break;
        case "toilets":
          // 2 Simple Toilets 46 m²: one exterior entrance per toilet module,
          // on opposite long facades as shown in the architecture plan.
          addDetailedDoor("front", 0.25, 0.13, 2.14, 0.08, "front-door");
          addDetailedDoor("back", 0.75, 0.13, 2.14, 0.08, "back-door");
          addFrontOpening(
            0.5,
            0.17,
            1.08,
            1.0,
            glassMaterial,
            "screened-opening",
          );
          break;
        case "toilets-ablution":
          // 2 Simple Toilets / Ablution 46 m²: again only two exterior
          // entrances total, one on each opposite long facade.
          addDetailedDoor("front", 0.25, 0.13, 2.14, 0.08, "front-door");
          addDetailedDoor("back", 0.75, 0.13, 2.14, 0.08, "back-door");
          addFrontOpening(
            0.5,
            0.19,
            1.15,
            0.95,
            glassMaterial,
            "ablution-screened-opening",
          );
          break;
        case "prayer": {
          // Prayer Room access must always be on the geographic EAST facade.
          // Local +X follows frame.longAxis and local +Z follows frame.shortAxis,
          // so choose whichever local facade has the strongest +world-X
          // (eastward) outward normal after the GIS footprint rotation.
          const eastFacingSide = [
            { side: "right", eastComponent: Number(frame.longAxis?.x) || 0 },
            { side: "left", eastComponent: -(Number(frame.longAxis?.x) || 0) },
            { side: "back", eastComponent: Number(frame.shortAxis?.x) || 0 },
            {
              side: "front",
              eastComponent: -(Number(frame.shortAxis?.x) || 0),
            },
          ].sort((a, b) => b.eastComponent - a.eastComponent)[0].side;

          addDetailedDoor(
            eastFacingSide,
            0.32,
            0.13,
            2.12,
            0.08,
            "east-door-a",
          );
          addDetailedDoor(
            eastFacingSide,
            0.68,
            0.13,
            2.12,
            0.08,
            "east-door-b",
          );
          break;
        }
        case "office":
          addFrontOpening(0.14, 0.11, 2.1, 0.08, doorMaterial, "door");
          addFrontOpening(0.4, 0.16, 1.08, 1.0, glassMaterial, "window-a");
          addFrontOpening(0.66, 0.16, 1.08, 1.0, glassMaterial, "window-b");
          addFrontOpening(0.86, 0.1, 1.08, 1.0, glassMaterial, "window-c");
          break;
        case "cycling":
          addFrontOpening(0.2, 0.12, 2.08, 0.08, doorMaterial, "door");
          addFrontOpening(
            0.56,
            0.26,
            1.16,
            0.98,
            glassMaterial,
            "display-window",
          );
          break;
        case "visitor":
        default:
          addFrontOpening(0.18, 0.15, 2.08, 0.08, doorMaterial, "door");
          addFrontOpening(0.58, 0.28, 1.14, 0.98, glassMaterial, "window");
          break;
      }

      return group;
    },

    isStandaloneHalfBasketballCourtStructure(properties = {}) {
      return this.getStandaloneAppModel(properties) === "half_basketball_court";
    },

    isStandaloneSkateparkStructure(properties = {}) {
      return this.getStandaloneAppModel(properties) === "skatepark";
    },

    isStandaloneFullBasketballCourtStructure(properties = {}) {
      const model = this.getStandaloneAppModel(properties);
      return (
        model === "basketball_court" ||
        model === "full_basketball_court" ||
        model === "basketball_courts" ||
        model === "basketball_courts_3" ||
        (model.includes("basketball") &&
          !model.includes("half") &&
          !model.includes("muga"))
      );
    },

    isStandaloneMugaCourtStructure(properties = {}) {
      const model = this.getStandaloneAppModel(properties);
      return (
        model === "muga_court" ||
        model === "muga_courts" ||
        /^muga(?:_|$)/.test(model)
      );
    },

    getStandaloneHalfBasketballHoopSide(properties = {}) {
      const side = String(
        this.getStandaloneCaseInsensitiveProperty(properties, [
          "APP_HOOP_SIDE",
          "app_hoop_side",
        ]) || "north",
      )
        .trim()
        .toLowerCase();

      return ["north", "east", "south", "west"].includes(side) ? side : "north";
    },

    getStandaloneSkateparkSlopeSide(properties = {}) {
      // APP_HOOP_SIDE is reused as a generic orientation field for special
      // sports models. For a skatepark it identifies the side containing the
      // long roll-in / slope that connects into the bowl.
      return this.getStandaloneHalfBasketballHoopSide(properties);
    },

    isStandalonePointInsidePolygonRing(point, ring = []) {
      if (
        !point ||
        !Array.isArray(ring) ||
        ring.length < 3 ||
        !Number.isFinite(Number(point.x)) ||
        !Number.isFinite(Number(point.y))
      ) {
        return false;
      }

      let inside = false;
      for (
        let currentIndex = 0, previousIndex = ring.length - 1;
        currentIndex < ring.length;
        previousIndex = currentIndex, currentIndex += 1
      ) {
        const current = ring[currentIndex];
        const previous = ring[previousIndex];
        const currentY = Number(current.y);
        const previousY = Number(previous.y);
        const crossesHorizontalRay =
          currentY > Number(point.y) !== previousY > Number(point.y);

        if (!crossesHorizontalRay) continue;

        const denominator = previousY - currentY;
        if (Math.abs(denominator) < 0.0000001) continue;

        const intersectionX =
          ((Number(previous.x) - Number(current.x)) *
            (Number(point.y) - currentY)) /
            denominator +
          Number(current.x);

        if (Number(point.x) < intersectionX) inside = !inside;
      }

      return inside;
    },

    getStandalonePolygonSupportHeight(localRings, modelData) {
      if (!Array.isArray(localRings) || !localRings.length) return 0;
      const outer = localRings[0];
      const holes = localRings.slice(1);
      if (!Array.isArray(outer) || outer.length < 3) return 0;

      const samples = [];
      const pushSample = (point) => {
        if (
          !point ||
          !Number.isFinite(Number(point.x)) ||
          !Number.isFinite(Number(point.y))
        ) {
          return;
        }
        samples.push({
          x: Number(point.x),
          z: Number(point.y),
        });
      };

      outer.forEach((point, index) => {
        const next = outer[(index + 1) % outer.length];
        pushSample(point);

        const edgeLength = point.distanceTo(next);
        const divisions = Math.max(
          1,
          Math.min(12, Math.ceil(edgeLength / 2.5)),
        );
        for (let step = 1; step < divisions; step += 1) {
          const ratio = step / divisions;
          pushSample({
            x: THREE.MathUtils.lerp(point.x, next.x, ratio),
            y: THREE.MathUtils.lerp(point.y, next.y, ratio),
          });
        }
      });

      const centroid = this.getStandalonePolygonCentroid(outer);
      if (centroid) pushSample(centroid);

      const bounds = this.getStandalonePolygonBounds(outer);
      if (bounds) {
        const columns = 7;
        const rows = 7;
        for (let column = 0; column < columns; column += 1) {
          const x = THREE.MathUtils.lerp(
            bounds.minX,
            bounds.maxX,
            (column + 0.5) / columns,
          );
          for (let row = 0; row < rows; row += 1) {
            const y = THREE.MathUtils.lerp(
              bounds.minY,
              bounds.maxY,
              (row + 0.5) / rows,
            );
            const candidate = new THREE.Vector2(x, y);
            if (!this.isStandalonePointInsidePolygonRing(candidate, outer)) {
              continue;
            }
            if (
              holes.some((hole) =>
                this.isStandalonePointInsidePolygonRing(candidate, hole),
              )
            ) {
              continue;
            }
            pushSample(candidate);
          }
        }
      }

      let highest = -Infinity;
      samples.forEach((point) => {
        const height = this.getStandaloneTerrainHeightAtPoint(point, modelData);
        if (Number.isFinite(Number(height))) {
          highest = Math.max(highest, Number(height));
        }
      });

      return Number.isFinite(highest) ? highest : 0;
    },

    getStandaloneDetailedEquipmentStyle(featureType = "", properties = {}) {
      const descriptor = this.getStandaloneStructureDescriptor(
        featureType,
        properties,
      );
      if (!descriptor) return null;

      if (/\brunning\s*track\b/.test(descriptor)) {
        return {
          key: "running-track-50m",
          kind: "running-track",
          label: "Running Track 50m",
        };
      }
      if (/\bpetanque\b|\bp[eé]tanque\b/.test(descriptor)) {
        return { key: "petanque", kind: "petanque", label: "Petanque" };
      }
      if (/\b(?:bicycle|bike)\s*rack\b/.test(descriptor)) {
        return {
          key: "bicycle-rack",
          kind: "bicycle-rack",
          label: "Bicycle Rack",
        };
      }
      if (
        /\btable\s*tenn?is\b|\btable\s*tenis\b|\bping\s*pong\b/.test(descriptor)
      ) {
        return {
          key: "table-tennis",
          kind: "table-tennis",
          label: "Table Tennis",
        };
      }

      // Outdoor Gym assets in the Qadisiyah GIS are labelled with a
      // zero-padded suffix (OUTDOOR GYM 01 ... OUTDOOR GYM 06). Accept both
      // padded and non-padded variants, plus the other workout naming forms.
      const workoutMatch =
        descriptor.match(
          /(?:workout\s*(?:equipment)?|outdoor\s*(?:gym|fitness)|fitness\s*(?:trail|station|equipment)|gym\s*(?:equipment)?)[^0-9]{0,20}(?:type\s*)?0?([1-6])\b/,
        ) || descriptor.match(/\btype\s*0?([1-6])\b/);
      if (workoutMatch) {
        const workoutType = Math.max(1, Math.min(6, Number(workoutMatch[1])));
        return {
          key: `workout-equipment-${workoutType}`,
          kind: "workout",
          workoutType,
          label: `Workout Equipment ${workoutType}`,
        };
      }

      return null;
    },

    createStandaloneDetailedEquipmentGroup(
      localRings,
      modelData,
      style,
      properties = {},
    ) {
      if (!style || !Array.isArray(localRings) || !localRings.length)
        return null;
      const outer = localRings[0];
      const frame = this.getStandaloneModularBuildingFrame(outer);
      if (!frame) return null;

      const group = new THREE.Group();
      group.name = `detailed-equipment-${style.key}`;
      group.userData.detailedEquipment = true;
      group.userData.detailedEquipmentKey = style.key;
      group.userData.detailedEquipmentLabel = style.label;

      const baseY =
        this.getStandalonePolygonSupportHeight(localRings, modelData) +
        THREE_STRUCTURE_SURFACE_CLEARANCE_METERS;
      group.position.set(frame.center.x, baseY, frame.center.y);
      group.rotation.y = frame.rotationY;

      const length = Math.max(1.2, frame.length * 0.94);
      const width = Math.max(0.8, frame.width * 0.94);
      const dark = new THREE.MeshStandardMaterial({
        color: 0x303638,
        emissive: 0x101415,
        emissiveIntensity: 0.08,
        roughness: 0.72,
        metalness: 0.3,
      });
      const cyan = new THREE.MeshStandardMaterial({
        color: 0x62d7d3,
        emissive: 0x123d3d,
        emissiveIntensity: 0.22,
        roughness: 0.52,
        metalness: 0.22,
      });
      const steel = new THREE.MeshStandardMaterial({
        color: 0xbfc8cb,
        emissive: 0x242a2c,
        emissiveIntensity: 0.08,
        roughness: 0.35,
        metalness: 0.76,
      });
      const white = new THREE.MeshStandardMaterial({
        color: 0xf2f3f0,
        roughness: 0.72,
        metalness: 0.02,
      });

      const addBox = (name, sx, sy, sz, x, y, z, material, rx = 0, rz = 0) => {
        const mesh = new THREE.Mesh(
          new THREE.BoxGeometry(sx, sy, sz),
          material,
        );
        mesh.name = name;
        mesh.position.set(x, y, z);
        mesh.rotation.x = rx;
        mesh.rotation.z = rz;
        mesh.renderOrder = 20;
        group.add(mesh);
        return mesh;
      };
      const addCylinder = (
        name,
        radius,
        height,
        x,
        y,
        z,
        material,
        radial = 12,
      ) => {
        const mesh = new THREE.Mesh(
          new THREE.CylinderGeometry(radius, radius, height, radial, 1),
          material,
        );
        mesh.name = name;
        mesh.position.set(x, y, z);
        mesh.renderOrder = 20;
        group.add(mesh);
        return mesh;
      };

      if (style.kind === "running-track") {
        const trackMat = new THREE.MeshStandardMaterial({
          color: 0xa63b32,
          emissive: 0x35100d,
          emissiveIntensity: 0.16,
          roughness: 0.9,
          metalness: 0.01,
        });
        addBox(
          "running-track-synthetic-rubber",
          length,
          0.12,
          width,
          0,
          0.08,
          0,
          trackMat,
        );
        const laneCount = 7;
        const laneMaterial = new THREE.LineBasicMaterial({
          color: 0xf7eee8,
          transparent: true,
          opacity: 0.92,
          depthWrite: false,
        });
        const lanePositions = [];
        for (let lane = 0; lane <= laneCount; lane += 1) {
          const z = -width / 2 + (width * lane) / laneCount;
          lanePositions.push(-length / 2, 0.151, z, length / 2, 0.151, z);
        }
        const laneGeo = new THREE.BufferGeometry();
        laneGeo.setAttribute(
          "position",
          new THREE.Float32BufferAttribute(lanePositions, 3),
        );
        const lanes = new THREE.LineSegments(laneGeo, laneMaterial);
        lanes.name = "running-track-lane-lines";
        lanes.renderOrder = 24;
        group.add(lanes);
      } else if (style.kind === "petanque") {
        const soilMat = new THREE.MeshStandardMaterial({
          color: 0xb9aa8b,
          emissive: 0x2d281f,
          emissiveIntensity: 0.08,
          roughness: 1,
        });
        const woodMat = new THREE.MeshStandardMaterial({
          color: 0x7b512d,
          emissive: 0x1d1008,
          emissiveIntensity: 0.06,
          roughness: 0.88,
        });
        addBox(
          "petanque-compacted-soil",
          length * 0.97,
          0.08,
          width * 0.91,
          0,
          0.055,
          0,
          soilMat,
        );
        const border = Math.min(0.16, width * 0.08);
        addBox(
          "petanque-border-left",
          length,
          0.24,
          border,
          0,
          0.12,
          -width / 2 + border / 2,
          woodMat,
        );
        addBox(
          "petanque-border-right",
          length,
          0.24,
          border,
          0,
          0.12,
          width / 2 - border / 2,
          woodMat,
        );
        addBox(
          "petanque-border-front",
          border,
          0.24,
          width,
          -length / 2 + border / 2,
          0.12,
          0,
          woodMat,
        );
        addBox(
          "petanque-border-back",
          border,
          0.24,
          width,
          length / 2 - border / 2,
          0.12,
          0,
          woodMat,
        );
        [-0.18, 0.08, 0.25].forEach((x, i) => {
          const ball = new THREE.Mesh(
            new THREE.SphereGeometry(0.055, 10, 8),
            i === 1 ? cyan : steel,
          );
          ball.position.set(
            x * Math.min(length, 4),
            0.13,
            (i - 1) * Math.min(width * 0.12, 0.2),
          );
          group.add(ball);
        });
      } else if (style.kind === "bicycle-rack") {
        const rackCount = Math.max(3, Math.min(7, Math.round(length / 0.75)));
        const spacing = Math.min(0.8, length / Math.max(1, rackCount));
        const bar = 0.055;
        for (let i = 0; i < rackCount; i += 1) {
          const x = (i - (rackCount - 1) / 2) * spacing;
          const rackH = Math.min(0.9, Math.max(0.62, width * 0.72));
          addCylinder(
            `bicycle-rack-post-a-${i}`,
            bar,
            rackH,
            x,
            rackH / 2,
            -0.32,
            steel,
            10,
          );
          addCylinder(
            `bicycle-rack-post-b-${i}`,
            bar,
            rackH,
            x,
            rackH / 2,
            0.32,
            steel,
            10,
          );
          const top = addCylinder(
            `bicycle-rack-top-${i}`,
            bar,
            0.64,
            x,
            rackH,
            0,
            steel,
            10,
          );
          top.rotation.x = Math.PI / 2;
        }
      } else if (style.kind === "table-tennis") {
        const tableL = Math.min(length * 0.9, 2.74);
        const tableW = Math.min(width * 0.9, 1.53);
        const topY = 0.76;
        const topMat = new THREE.MeshStandardMaterial({
          color: 0x73797b,
          emissive: 0x16191a,
          emissiveIntensity: 0.08,
          roughness: 0.72,
          metalness: 0.05,
        });
        addBox("table-tennis-top", tableL, 0.075, tableW, 0, topY, 0, topMat);
        addBox(
          "table-tennis-center-line",
          tableL,
          0.012,
          0.018,
          0,
          topY + 0.045,
          0,
          white,
        );
        const net = addBox(
          "table-tennis-net",
          0.035,
          0.16,
          tableW * 0.98,
          0,
          topY + 0.12,
          0,
          white,
        );
        net.material = new THREE.MeshStandardMaterial({
          color: 0xe7eeee,
          roughness: 0.5,
          metalness: 0.15,
        });
        [-1, 1].forEach((side) => {
          addBox(
            `table-tennis-foot-${side}`,
            tableL * 0.34,
            0.08,
            0.1,
            side * tableL * 0.24,
            0.09,
            0,
            dark,
          );
          const legA = addBox(
            `table-tennis-leg-a-${side}`,
            0.08,
            0.9,
            0.08,
            side * tableL * 0.28,
            0.39,
            -tableW * 0.22,
            dark,
            0,
            side * 0.48,
          );
          const legB = addBox(
            `table-tennis-leg-b-${side}`,
            0.08,
            0.9,
            0.08,
            side * tableL * 0.28,
            0.39,
            tableW * 0.22,
            dark,
            0,
            side * 0.48,
          );
          void legA;
          void legB;
        });
      } else if (style.kind === "workout") {
        const type = Number(style.workoutType) || 1;

        // The six Qadisiyah outdoor-gym assets are point features, so use the
        // product-sheet dimensions rather than stretching the equipment to the
        // GIS marker footprint.
        const designLength =
          {
            1: 2.14,
            2: 1.8,
            3: 3.1,
            4: 1.72,
            5: 1.78,
            6: 1.24,
          }[type] || 2.0;
        const designWidth =
          {
            1: 0.9,
            2: 0.6,
            3: 0.9,
            4: 1.55,
            5: 0.82,
            6: 0.78,
          }[type] || 0.9;

        const unitL = Math.min(length * 0.92, designLength);
        const unitW = Math.min(width * 0.9, designWidth);
        const postH = 1.72;

        const addTube = (name, points, radius, material) => {
          const curve = new THREE.CatmullRomCurve3(
            points.map((p) => new THREE.Vector3(p[0], p[1], p[2])),
          );
          const mesh = new THREE.Mesh(
            new THREE.TubeGeometry(
              curve,
              Math.max(8, points.length * 5),
              radius,
              8,
              false,
            ),
            material,
          );
          mesh.name = name;
          mesh.renderOrder = 22;
          group.add(mesh);
          return mesh;
        };

        const addFlatCapsule = (
          name,
          capsuleLength,
          capsuleWidth,
          height,
          x,
          y,
          z,
          material,
          rotY = 0,
        ) => {
          const radius = capsuleWidth / 2;
          const straight = Math.max(0.02, capsuleLength - capsuleWidth);
          const shape = new THREE.Shape();
          shape.moveTo(-straight / 2, -radius);
          shape.lineTo(straight / 2, -radius);
          shape.absarc(
            straight / 2,
            0,
            radius,
            -Math.PI / 2,
            Math.PI / 2,
            false,
          );
          shape.lineTo(-straight / 2, radius);
          shape.absarc(
            -straight / 2,
            0,
            radius,
            Math.PI / 2,
            Math.PI * 1.5,
            false,
          );
          const geo = new THREE.ExtrudeGeometry(shape, {
            depth: height,
            bevelEnabled: false,
            curveSegments: 10,
          });
          geo.rotateX(-Math.PI / 2);
          geo.translate(0, -height / 2, 0);
          const mesh = new THREE.Mesh(geo, material);
          mesh.name = name;
          mesh.position.set(x, y, z);
          mesh.rotation.y = rotY;
          mesh.renderOrder = 21;
          group.add(mesh);
          return mesh;
        };

        const addSign = (prefix, x, z, h = postH) => {
          addCylinder(`${prefix}-post`, 0.045, h, x, h / 2, z, dark, 12);
          const disc = new THREE.Mesh(
            new THREE.CylinderGeometry(0.19, 0.19, 0.045, 20),
            steel,
          );
          disc.name = `${prefix}-instruction-disc`;
          disc.rotation.z = Math.PI / 2;
          disc.position.set(x, h - 0.16, z);
          disc.renderOrder = 22;
          group.add(disc);
        };

        if (type === 1) {
          // TYPE 1 — five stepping pods with alternating heights and the
          // turquoise guidance rail shown in the design sheet.
          const xs = [-0.43, -0.215, 0, 0.215, 0.43].map((v) => v * unitL);
          xs.forEach((x, i) => {
            const h = 0.18 + (i % 2) * 0.055;
            addCylinder(
              `workout-1-support-${i}`,
              0.045,
              h,
              x,
              h / 2,
              0,
              cyan,
              10,
            );
            addFlatCapsule(
              `workout-1-pad-${i}`,
              Math.min(0.34, unitL * 0.17),
              Math.min(0.26, unitW * 0.3),
              0.07,
              x,
              h + 0.035,
              0,
              dark,
            );
          });
          const sx = -unitL * 0.48;
          const sz = -unitW * 0.42;
          addSign("workout-1-sign", sx, sz);
          addTube(
            "workout-1-guide-rail",
            [
              [sx + 0.02, 1.18, sz + 0.03],
              [-unitL * 0.26, 1.32, -unitW * 0.28],
              [unitL * 0.12, 1.08, -unitW * 0.18],
              [unitL * 0.43, 0.58, -unitW * 0.06],
            ],
            0.035,
            cyan,
          );
        } else if (type === 2) {
          // TYPE 2 — narrow raised balance beam, turquoise feet and a small
          // U-shaped grab rail at the entry end.
          const beam = addBox(
            "workout-2-balance-deck",
            unitL,
            0.1,
            unitW * 0.72,
            0,
            0.3,
            0,
            dark,
          );
          beam.rotation.z = -0.035;
          [-0.42, 0.42].forEach((v, i) => {
            addCylinder(
              `workout-2-foot-${i}`,
              0.045,
              0.27,
              v * unitL,
              0.135,
              0,
              cyan,
              10,
            );
          });
          const hx = -unitL * 0.38;
          addTube(
            "workout-2-entry-handle",
            [
              [hx, 0.37, -unitW * 0.26],
              [hx, 0.78, -unitW * 0.26],
              [hx + unitL * 0.16, 0.85, -unitW * 0.26],
              [hx + unitL * 0.2, 0.6, -unitW * 0.26],
            ],
            0.035,
            cyan,
          );
        } else if (type === 3) {
          // TYPE 3 — inclined training ramp with four turquoise guard-rail
          // posts per side and the central product-information post.
          const deck = addBox(
            "workout-3-ramp-deck",
            unitL,
            0.1,
            unitW * 0.72,
            0,
            0.26,
            0,
            white,
          );
          deck.rotation.z = -0.075;
          [-1, 1].forEach((side) => {
            const z = side * unitW * 0.4;
            const railPts = [];
            [-0.44, -0.15, 0.15, 0.44].forEach((v, i) => {
              const x = v * unitL;
              const baseY = 0.32 - v * 0.2;
              addCylinder(
                `workout-3-rail-post-${side}-${i}`,
                0.035,
                0.76,
                x,
                baseY + 0.38,
                z,
                cyan,
                10,
              );
              railPts.push([x, baseY + 0.76, z]);
            });
            addTube(`workout-3-top-rail-${side}`, railPts, 0.035, cyan);
          });
          addSign("workout-3-sign", unitL * 0.39, 0);
        } else if (type === 4) {
          // TYPE 4 — three elongated anti-slip stepping platforms in the
          // triangular arrangement shown in plan (not circular pads).
          const padL = Math.min(0.66, unitL * 0.4);
          const padW = Math.min(0.34, unitW * 0.27);
          const pads = [
            [-unitL * 0.22, unitW * 0.28, 0],
            [unitL * 0.25, unitW * 0.22, 0],
            [unitL * 0.08, -unitW * 0.3, Math.PI / 2],
          ];
          pads.forEach((p, i) => {
            const supportH = 0.22 + (i === 2 ? 0.06 : 0);
            addCylinder(
              `workout-4-support-${i}`,
              0.045,
              supportH,
              p[0],
              supportH / 2,
              p[1],
              cyan,
              10,
            );
            addFlatCapsule(
              `workout-4-platform-${i}`,
              padL,
              padW,
              0.075,
              p[0],
              supportH + 0.038,
              p[1],
              dark,
              p[2],
            );
          });
          addSign("workout-4-sign", -unitL * 0.43, -unitW * 0.37);
        } else if (type === 5) {
          // TYPE 5 — seated twisting/press station: rear instruction post,
          // turquoise curved mechanism, dark seat and front handle loop.
          const px = -unitL * 0.36;
          addSign("workout-5-sign", px, 0);
          addCylinder(
            "workout-5-seat-column",
            0.055,
            0.48,
            unitL * 0.04,
            0.24,
            0,
            cyan,
            10,
          );
          addFlatCapsule(
            "workout-5-seat",
            0.48,
            0.34,
            0.09,
            unitL * 0.04,
            0.52,
            0,
            dark,
          );
          addTube(
            "workout-5-frame",
            [
              [unitL * 0.18, 0.18, 0],
              [unitL * 0.26, 0.62, 0],
              [unitL * 0.34, 1.04, 0],
              [unitL * 0.43, 1.18, 0],
            ],
            0.04,
            cyan,
          );
          addTube(
            "workout-5-handle-loop",
            [
              [unitL * 0.37, 0.96, -unitW * 0.2],
              [unitL * 0.49, 1.02, -unitW * 0.2],
              [unitL * 0.49, 1.02, unitW * 0.2],
              [unitL * 0.37, 0.96, unitW * 0.2],
            ],
            0.033,
            cyan,
          );
          addCylinder(
            "workout-5-pivot",
            0.12,
            0.12,
            unitL * 0.25,
            0.7,
            0,
            dark,
            18,
          ).rotation.x = Math.PI / 2;
        } else {
          // TYPE 6 — seated pedal trainer with the crank/pedal assembly in
          // front and the tall information post behind the seat.
          const px = -unitL * 0.38;
          addSign("workout-6-sign", px, 0);
          addCylinder(
            "workout-6-seat-column",
            0.05,
            0.44,
            -unitL * 0.03,
            0.22,
            0,
            cyan,
            10,
          );
          addFlatCapsule(
            "workout-6-seat",
            0.44,
            0.34,
            0.09,
            -unitL * 0.03,
            0.49,
            0,
            dark,
          );
          addTube(
            "workout-6-back-frame",
            [
              [-unitL * 0.16, 0.45, -unitW * 0.18],
              [-unitL * 0.2, 0.83, -unitW * 0.18],
              [-unitL * 0.2, 0.83, unitW * 0.18],
              [-unitL * 0.16, 0.45, unitW * 0.18],
            ],
            0.035,
            cyan,
          );
          const crankX = unitL * 0.3;
          const crank = addCylinder(
            "workout-6-crank-hub",
            0.14,
            0.12,
            crankX,
            0.34,
            0,
            cyan,
            18,
          );
          crank.rotation.x = Math.PI / 2;
          [-1, 1].forEach((s) => {
            const arm = addBox(
              `workout-6-crank-arm-${s}`,
              0.25,
              0.035,
              0.045,
              crankX + s * 0.11,
              0.34 + s * 0.1,
              0,
              dark,
              0,
              s * 0.7,
            );
            void arm;
            addBox(
              `workout-6-pedal-${s}`,
              0.18,
              0.045,
              0.11,
              crankX + s * 0.22,
              0.34 + s * 0.18,
              0,
              dark,
            );
          });
          addTube(
            "workout-6-front-frame",
            [
              [unitL * 0.15, 0.14, 0],
              [unitL * 0.23, 0.42, 0],
              [unitL * 0.31, 0.62, 0],
            ],
            0.035,
            cyan,
          );
        }
      } else {
        return null;
      }

      return group;
    },

    createStandaloneDetailedEquipmentPointGroup(local, modelData, style) {
      if (!local || !style) return null;
      // Point assets receive a small design footprint so the same detailed
      // renderer can be used even when ArcGIS stores the equipment as points.
      const defaults = {
        "bicycle-rack": [3.2, 1.4],
        "table-tennis": [3.1, 1.9],
        workout: [3.2, 2.0],
        petanque: [15, 4],
        "running-track": [50, 8.54],
      };
      const dims = defaults[style.kind] || [2.5, 1.5];
      const halfL = dims[0] / 2;
      const halfW = dims[1] / 2;
      const rings = [
        [
          new THREE.Vector2(local.x - halfL, local.z - halfW),
          new THREE.Vector2(local.x + halfL, local.z - halfW),
          new THREE.Vector2(local.x + halfL, local.z + halfW),
          new THREE.Vector2(local.x - halfL, local.z + halfW),
        ],
      ];
      return this.createStandaloneDetailedEquipmentGroup(
        rings,
        modelData,
        style,
        {},
      );
    },

    getStandaloneHalfBasketballCourtFrame(outer, hoopSide = "north") {
      if (!Array.isArray(outer) || outer.length < 3) return null;
      const centroid = this.getStandalonePolygonCentroid(outer);
      if (!centroid) return null;

      let xx = 0;
      let xy = 0;
      let yy = 0;
      outer.forEach((point) => {
        const dx = Number(point.x) - centroid.x;
        const dy = Number(point.y) - centroid.y;
        xx += dx * dx;
        xy += dx * dy;
        yy += dy * dy;
      });

      const angle = 0.5 * Math.atan2(2 * xy, xx - yy);
      const principal = new THREE.Vector2(
        Math.cos(angle),
        Math.sin(angle),
      ).normalize();
      const secondary = new THREE.Vector2(
        -principal.y,
        principal.x,
      ).normalize();

      // Local x is east and local polygon y is world z, where north is
      // negative z because lngLatToTerrainMeters negates latitude.
      const targetVectors = {
        north: new THREE.Vector2(0, -1),
        east: new THREE.Vector2(1, 0),
        south: new THREE.Vector2(0, 1),
        west: new THREE.Vector2(-1, 0),
      };
      const target = targetVectors[hoopSide] || targetVectors.north;
      const forward =
        Math.abs(principal.dot(target)) >= Math.abs(secondary.dot(target))
          ? principal.clone()
          : secondary.clone();

      if (forward.dot(target) < 0) forward.multiplyScalar(-1);

      // A Three.js group's local +Z axis will point along forward. After the
      // Y rotation below, its local +X axis becomes this width direction.
      const widthAxis = new THREE.Vector2(forward.y, -forward.x).normalize();

      let minForward = Infinity;
      let maxForward = -Infinity;
      let minWidth = Infinity;
      let maxWidth = -Infinity;
      outer.forEach((point) => {
        const relative = new THREE.Vector2(
          Number(point.x) - centroid.x,
          Number(point.y) - centroid.y,
        );
        const forwardProjection = relative.dot(forward);
        const widthProjection = relative.dot(widthAxis);
        minForward = Math.min(minForward, forwardProjection);
        maxForward = Math.max(maxForward, forwardProjection);
        minWidth = Math.min(minWidth, widthProjection);
        maxWidth = Math.max(maxWidth, widthProjection);
      });

      const forwardCenter = (minForward + maxForward) / 2;
      const widthCenter = (minWidth + maxWidth) / 2;
      const center = centroid
        .clone()
        .add(forward.clone().multiplyScalar(forwardCenter))
        .add(widthAxis.clone().multiplyScalar(widthCenter));

      return {
        center,
        forward,
        widthAxis,
        rotationY: Math.atan2(forward.x, forward.y),
        length: Math.max(2, (maxForward - minForward) * 0.92),
        width: Math.max(2, (maxWidth - minWidth) * 0.92),
      };
    },

    createStandaloneHalfBasketballCourtGroup(
      localRings,
      modelData,
      properties = {},
    ) {
      if (!Array.isArray(localRings) || !localRings.length) return null;
      const outer = localRings[0];
      const holes = localRings.slice(1);
      if (!Array.isArray(outer) || outer.length < 3) return null;

      const hoopSide = this.getStandaloneHalfBasketballHoopSide(properties);
      const frame = this.getStandaloneHalfBasketballCourtFrame(outer, hoopSide);
      if (!frame) return null;

      const group = new THREE.Group();
      group.name = "half-basketball-court";
      group.userData.appModel = "half_basketball_court";
      group.userData.hoopSide = hoopSide;

      const terrainSupportY = this.getStandalonePolygonSupportHeight(
        localRings,
        modelData,
      );
      const deckTopY = terrainSupportY + 0.22;

      const contour = outer.map((point) => new THREE.Vector2(point.x, point.y));
      const holeVectors = holes.map((ring) =>
        ring.map((point) => new THREE.Vector2(point.x, point.y)),
      );
      const flattened = contour.concat(...holeVectors);
      const faces = THREE.ShapeUtils.triangulateShape(contour, holeVectors);
      const deckPositions = [];
      const deckOutlinePositions = [];

      faces.forEach((face) => {
        face.forEach((index) => {
          const point = flattened[index];
          if (point) deckPositions.push(point.x, deckTopY, point.y);
        });
      });

      const appendWallRing = (ring) => {
        ring.forEach((point, index) => {
          const next = ring[(index + 1) % ring.length];
          const baseA =
            this.getStandaloneTerrainHeightAtPoint(
              { x: point.x, z: point.y },
              modelData,
            ) + 0.025;
          const baseB =
            this.getStandaloneTerrainHeightAtPoint(
              { x: next.x, z: next.y },
              modelData,
            ) + 0.025;

          deckPositions.push(
            point.x,
            deckTopY,
            point.y,
            point.x,
            baseA,
            point.y,
            next.x,
            baseB,
            next.y,

            point.x,
            deckTopY,
            point.y,
            next.x,
            baseB,
            next.y,
            next.x,
            deckTopY,
            next.y,
          );

          deckOutlinePositions.push(
            point.x,
            deckTopY + 0.025,
            point.y,
            next.x,
            deckTopY + 0.025,
            next.y,
          );
        });
      };
      appendWallRing(outer);
      holes.forEach(appendWallRing);

      if (deckPositions.length) {
        const geometry = new THREE.BufferGeometry();
        geometry.setAttribute(
          "position",
          new THREE.Float32BufferAttribute(deckPositions, 3),
        );
        geometry.computeVertexNormals();
        geometry.computeBoundingSphere();

        const material = new THREE.MeshStandardMaterial({
          color: 0x176ea6,
          emissive: 0x073854,
          emissiveIntensity: 0.38,
          roughness: 0.78,
          metalness: 0.02,
          transparent: true,
          opacity: 0.96,
          side: THREE.DoubleSide,
          depthWrite: true,
          polygonOffset: true,
          polygonOffsetFactor: -4,
          polygonOffsetUnits: -8,
        });
        const deck = new THREE.Mesh(geometry, material);
        deck.name = "half-basketball-court-deck";
        deck.renderOrder = 14;
        deck.frustumCulled = false;
        group.add(deck);
      }

      if (deckOutlinePositions.length) {
        const geometry = new THREE.BufferGeometry();
        geometry.setAttribute(
          "position",
          new THREE.Float32BufferAttribute(deckOutlinePositions, 3),
        );
        const material = new THREE.LineBasicMaterial({
          color: 0xeafcff,
          transparent: true,
          opacity: 0.96,
          depthWrite: false,
        });
        const outline = new THREE.LineSegments(geometry, material);
        outline.name = "half-basketball-court-footprint-outline";
        outline.renderOrder = 18;
        outline.frustumCulled = false;
        group.add(outline);
      }

      const details = new THREE.Group();
      details.name = "half-basketball-court-details";
      details.position.set(frame.center.x, deckTopY, frame.center.y);
      details.rotation.y = frame.rotationY;

      const courtLength = frame.length;
      const courtWidth = frame.width;
      const halfLength = courtLength / 2;
      const halfWidth = courtWidth / 2;
      const baselineInset = Math.max(0.25, Math.min(0.55, courtLength * 0.035));
      const baselineZ = halfLength - baselineInset;
      const laneLength = Math.max(2.8, Math.min(5.8, courtLength * 0.43));
      const laneWidth = Math.max(2.4, Math.min(4.9, courtWidth * 0.38));
      const laneEndZ = baselineZ - laneLength;
      const backboardZ =
        baselineZ - Math.max(0.75, Math.min(1.2, courtLength * 0.075));
      const rimZ =
        backboardZ - Math.max(0.32, Math.min(0.46, courtLength * 0.03));

      const laneMaterial = new THREE.MeshBasicMaterial({
        // Dark violet acrylic paint from the half-court detailed design.
        color: 0x4b3b6b,
        transparent: true,
        opacity: 0.84,
        side: THREE.DoubleSide,
        depthWrite: false,
        polygonOffset: true,
        polygonOffsetFactor: -6,
        polygonOffsetUnits: -10,
      });
      const lane = new THREE.Mesh(
        new THREE.PlaneGeometry(laneWidth, laneLength),
        laneMaterial,
      );
      lane.name = "half-basketball-court-painted-key";
      lane.rotation.x = -Math.PI / 2;
      lane.position.set(0, 0.035, (baselineZ + laneEndZ) / 2);
      lane.renderOrder = 16;
      details.add(lane);

      const markingPositions = [];
      const pushLine = (x1, z1, x2, z2) => {
        markingPositions.push(x1, 0.065, z1, x2, 0.065, z2);
      };
      const pushArc = (
        centerX,
        centerZ,
        radius,
        startAngle,
        endAngle,
        segments = 28,
      ) => {
        let previous = null;
        for (let index = 0; index <= segments; index += 1) {
          const angle = THREE.MathUtils.lerp(
            startAngle,
            endAngle,
            index / segments,
          );
          const point = {
            x: centerX + Math.cos(angle) * radius,
            z: centerZ + Math.sin(angle) * radius,
          };
          if (previous) {
            pushLine(previous.x, previous.z, point.x, point.z);
          }
          previous = point;
        }
      };

      // Court boundary.
      pushLine(-halfWidth, -halfLength, halfWidth, -halfLength);
      pushLine(halfWidth, -halfLength, halfWidth, baselineZ);
      pushLine(halfWidth, baselineZ, -halfWidth, baselineZ);
      pushLine(-halfWidth, baselineZ, -halfWidth, -halfLength);

      // Painted key and free-throw line.
      pushLine(-laneWidth / 2, baselineZ, -laneWidth / 2, laneEndZ);
      pushLine(laneWidth / 2, baselineZ, laneWidth / 2, laneEndZ);
      pushLine(-laneWidth / 2, laneEndZ, laneWidth / 2, laneEndZ);

      const freeThrowRadius = Math.max(1.0, Math.min(1.8, laneWidth * 0.36));
      pushArc(0, laneEndZ, freeThrowRadius, 0, Math.PI * 2, 32);

      const restrictedRadius = Math.max(0.75, Math.min(1.25, laneWidth * 0.26));
      pushArc(0, rimZ, restrictedRadius, Math.PI, Math.PI * 2, 20);

      const threePointRadius = Math.max(
        2.6,
        Math.min(6.75, halfWidth - 0.4, courtLength * 0.48),
      );
      pushArc(0, rimZ, threePointRadius, Math.PI, Math.PI * 2, 36);

      // Backboard reference line and a short center tick on the open end.
      pushLine(-0.9, backboardZ, 0.9, backboardZ);
      pushLine(0, -halfLength, 0, -halfLength + 0.55);

      if (markingPositions.length) {
        const geometry = new THREE.BufferGeometry();
        geometry.setAttribute(
          "position",
          new THREE.Float32BufferAttribute(markingPositions, 3),
        );
        const material = new THREE.LineBasicMaterial({
          color: 0xf7ffff,
          transparent: true,
          opacity: 0.98,
          depthWrite: false,
        });
        const markings = new THREE.LineSegments(geometry, material);
        markings.name = "half-basketball-court-markings";
        markings.renderOrder = 19;
        details.add(markings);
      }

      const poleMaterial = new THREE.MeshStandardMaterial({
        color: 0x7135a8,
        emissive: 0x2b1048,
        emissiveIntensity: 0.35,
        roughness: 0.55,
        metalness: 0.18,
      });
      const backboardMaterial = new THREE.MeshStandardMaterial({
        color: 0xeafcff,
        emissive: 0x4c8994,
        emissiveIntensity: 0.18,
        roughness: 0.38,
        metalness: 0.05,
        transparent: true,
        opacity: 0.84,
      });
      const rimMaterial = new THREE.MeshStandardMaterial({
        color: 0xf06f24,
        emissive: 0x7c2608,
        emissiveIntensity: 0.42,
        roughness: 0.48,
        metalness: 0.12,
      });

      const poleZ = baselineZ - 0.2;
      const poleHeight = 3.5;
      const pole = new THREE.Mesh(
        new THREE.CylinderGeometry(0.11, 0.15, poleHeight, 10, 1),
        poleMaterial,
      );
      pole.name = "half-basketball-court-hoop-pole";
      pole.position.set(0, poleHeight / 2, poleZ);
      pole.renderOrder = 17;
      details.add(pole);

      const supportLength = Math.max(0.2, Math.abs(poleZ - backboardZ));
      const support = new THREE.Mesh(
        new THREE.BoxGeometry(0.13, 0.13, supportLength),
        poleMaterial,
      );
      support.name = "half-basketball-court-hoop-support";
      support.position.set(0, 3.45, (poleZ + backboardZ) / 2);
      support.renderOrder = 17;
      details.add(support);

      const backboard = new THREE.Mesh(
        new THREE.BoxGeometry(1.8, 1.05, 0.08),
        backboardMaterial,
      );
      backboard.name = "half-basketball-court-backboard";
      backboard.position.set(0, 3.55, backboardZ);
      backboard.renderOrder = 18;
      details.add(backboard);

      const rim = new THREE.Mesh(
        new THREE.TorusGeometry(0.23, 0.035, 8, 28),
        rimMaterial,
      );
      rim.name = "half-basketball-court-rim";
      rim.rotation.x = Math.PI / 2;
      rim.position.set(0, 3.05, rimZ);
      rim.renderOrder = 19;
      details.add(rim);

      const net = new THREE.Mesh(
        new THREE.CylinderGeometry(0.22, 0.14, 0.48, 12, 1, true),
        new THREE.MeshBasicMaterial({
          color: 0xf5ffff,
          wireframe: true,
          transparent: true,
          opacity: 0.74,
          depthWrite: false,
        }),
      );
      net.name = "half-basketball-court-net";
      net.position.set(0, 2.79, rimZ);
      net.renderOrder = 19;
      details.add(net);

      group.add(details);
      return group;
    },

    getStandaloneRectangularCourtFrame(outer) {
      if (!Array.isArray(outer) || outer.length < 3) return null;

      let best = null;
      const epsilon = 0.000001;

      for (let edgeIndex = 0; edgeIndex < outer.length; edgeIndex += 1) {
        const first = outer[edgeIndex];
        const second = outer[(edgeIndex + 1) % outer.length];
        const dx = Number(second.x) - Number(first.x);
        const dz = Number(second.y) - Number(first.y);
        const edgeLength = Math.hypot(dx, dz);
        if (edgeLength < epsilon) continue;

        const axisA = new THREE.Vector2(dx / edgeLength, dz / edgeLength);
        const axisB = new THREE.Vector2(-axisA.y, axisA.x);
        let minA = Infinity;
        let maxA = -Infinity;
        let minB = Infinity;
        let maxB = -Infinity;

        outer.forEach((point) => {
          const vector = new THREE.Vector2(Number(point.x), Number(point.y));
          const a = vector.dot(axisA);
          const b = vector.dot(axisB);
          minA = Math.min(minA, a);
          maxA = Math.max(maxA, a);
          minB = Math.min(minB, b);
          maxB = Math.max(maxB, b);
        });

        const spanA = maxA - minA;
        const spanB = maxB - minB;
        const area = spanA * spanB;
        if (!best || area < best.area) {
          best = { axisA, axisB, spanA, spanB, area };
        }
      }

      if (!best) return null;

      let forward =
        best.spanA >= best.spanB ? best.axisA.clone() : best.axisB.clone();

      // The court is symmetric end-to-end, so the sign is only made stable to
      // avoid the model flipping 180 degrees between reloads.
      if (
        forward.y > 0.000001 ||
        (Math.abs(forward.y) <= 0.000001 && forward.x < 0)
      ) {
        forward.multiplyScalar(-1);
      }

      const widthAxis = new THREE.Vector2(forward.y, -forward.x).normalize();
      let minForward = Infinity;
      let maxForward = -Infinity;
      let minWidth = Infinity;
      let maxWidth = -Infinity;

      outer.forEach((point) => {
        const vector = new THREE.Vector2(Number(point.x), Number(point.y));
        const pf = vector.dot(forward);
        const pw = vector.dot(widthAxis);
        minForward = Math.min(minForward, pf);
        maxForward = Math.max(maxForward, pf);
        minWidth = Math.min(minWidth, pw);
        maxWidth = Math.max(maxWidth, pw);
      });

      const centerForward = (minForward + maxForward) / 2;
      const centerWidth = (minWidth + maxWidth) / 2;
      const center = forward
        .clone()
        .multiplyScalar(centerForward)
        .add(widthAxis.clone().multiplyScalar(centerWidth));

      return {
        center,
        forward,
        widthAxis,
        rotationY: Math.atan2(forward.x, forward.y),
        length: Math.max(2, maxForward - minForward),
        width: Math.max(2, maxWidth - minWidth),
      };
    },

    createStandaloneSportCourtBase(
      localRings,
      modelData,
      {
        name = "sport-court",
        color = 0x9c9c9c,
        emissive = 0x151515,
        outlineColor = 0xf5ffff,
        slabLift = 0.1,
      } = {},
    ) {
      if (!Array.isArray(localRings) || !localRings.length) return null;
      const outer = localRings[0];
      const holes = localRings.slice(1);
      if (!Array.isArray(outer) || outer.length < 3) return null;

      const terrainSupportY = this.getStandalonePolygonSupportHeight(
        localRings,
        modelData,
      );
      const deckTopY = terrainSupportY + slabLift;
      const contour = outer.map((point) => new THREE.Vector2(point.x, point.y));
      const holeVectors = holes.map((ring) =>
        ring.map((point) => new THREE.Vector2(point.x, point.y)),
      );
      const flattened = contour.concat(...holeVectors);
      const faces = THREE.ShapeUtils.triangulateShape(contour, holeVectors);
      const positions = [];
      const outlinePositions = [];

      faces.forEach((face) => {
        face.forEach((index) => {
          const point = flattened[index];
          if (point) positions.push(point.x, deckTopY, point.y);
        });
      });

      const appendWallRing = (ring) => {
        ring.forEach((point, index) => {
          const next = ring[(index + 1) % ring.length];
          const baseA =
            this.getStandaloneTerrainHeightAtPoint(
              { x: point.x, z: point.y },
              modelData,
            ) + 0.02;
          const baseB =
            this.getStandaloneTerrainHeightAtPoint(
              { x: next.x, z: next.y },
              modelData,
            ) + 0.02;

          positions.push(
            point.x,
            deckTopY,
            point.y,
            point.x,
            baseA,
            point.y,
            next.x,
            baseB,
            next.y,

            point.x,
            deckTopY,
            point.y,
            next.x,
            baseB,
            next.y,
            next.x,
            deckTopY,
            next.y,
          );

          outlinePositions.push(
            point.x,
            deckTopY + 0.02,
            point.y,
            next.x,
            deckTopY + 0.02,
            next.y,
          );
        });
      };

      appendWallRing(outer);
      holes.forEach(appendWallRing);

      const group = new THREE.Group();
      group.name = `${name}-base`;

      if (positions.length) {
        const geometry = new THREE.BufferGeometry();
        geometry.setAttribute(
          "position",
          new THREE.Float32BufferAttribute(positions, 3),
        );
        geometry.computeVertexNormals();
        geometry.computeBoundingSphere();
        const material = new THREE.MeshStandardMaterial({
          color,
          emissive,
          emissiveIntensity: 0.12,
          roughness: 0.82,
          metalness: 0.01,
          side: THREE.DoubleSide,
          depthWrite: true,
          polygonOffset: true,
          polygonOffsetFactor: -4,
          polygonOffsetUnits: -8,
        });
        const mesh = new THREE.Mesh(geometry, material);
        mesh.name = `${name}-surface`;
        mesh.renderOrder = 14;
        mesh.frustumCulled = false;
        group.add(mesh);
      }

      if (outlinePositions.length) {
        const geometry = new THREE.BufferGeometry();
        geometry.setAttribute(
          "position",
          new THREE.Float32BufferAttribute(outlinePositions, 3),
        );
        const material = new THREE.LineBasicMaterial({
          color: outlineColor,
          transparent: true,
          opacity: 0.72,
          depthWrite: false,
        });
        const lines = new THREE.LineSegments(geometry, material);
        lines.name = `${name}-outline`;
        lines.renderOrder = 18;
        lines.frustumCulled = false;
        group.add(lines);
      }

      return { group, deckTopY };
    },

    createStandaloneBasketballMarkings(
      courtLength,
      courtWidth,
      {
        namePrefix = "basketball-court",
        color = 0xffffff,
        opacity = 0.98,
        y = 0.055,
        includeBoundary = true,
      } = {},
    ) {
      const length = Math.max(6, Number(courtLength) || 0);
      const width = Math.max(4, Number(courtWidth) || 0);
      const halfLength = length / 2;
      const halfWidth = width / 2;
      const positions = [];
      const pushLine = (x1, z1, x2, z2) => {
        positions.push(x1, y, z1, x2, y, z2);
      };
      const pushArc = (
        centerX,
        centerZ,
        radius,
        startAngle,
        endAngle,
        segments = 32,
      ) => {
        let previous = null;
        for (let index = 0; index <= segments; index += 1) {
          const angle = THREE.MathUtils.lerp(
            startAngle,
            endAngle,
            index / segments,
          );
          const point = {
            x: centerX + Math.cos(angle) * radius,
            z: centerZ + Math.sin(angle) * radius,
          };
          if (previous) pushLine(previous.x, previous.z, point.x, point.z);
          previous = point;
        }
      };

      const boundaryInset = Math.max(0.08, Math.min(0.28, width * 0.012));
      const xEdge = Math.max(0.2, halfWidth - boundaryInset);
      const zEdge = Math.max(0.3, halfLength - boundaryInset);

      if (includeBoundary) {
        pushLine(-xEdge, -zEdge, xEdge, -zEdge);
        pushLine(xEdge, -zEdge, xEdge, zEdge);
        pushLine(xEdge, zEdge, -xEdge, zEdge);
        pushLine(-xEdge, zEdge, -xEdge, -zEdge);
      }

      // Center line and center circle.
      pushLine(-xEdge, 0, xEdge, 0);
      const centerRadius = Math.max(0.9, Math.min(1.8, width * 0.12));
      pushArc(0, 0, centerRadius, 0, Math.PI * 2, 40);

      const laneWidth = Math.max(2.4, Math.min(4.9, width * 0.34));
      const laneDepth = Math.max(2.8, Math.min(5.8, length * 0.21));
      const freeThrowRadius = Math.max(0.9, Math.min(1.8, laneWidth * 0.37));
      const basketInset = Math.max(0.75, Math.min(1.58, length * 0.06));
      const threePointRadius = Math.max(
        2.6,
        Math.min(6.75, halfWidth - 0.35, length * 0.26),
      );
      const restrictedRadius = Math.max(0.7, Math.min(1.25, laneWidth * 0.26));

      [-1, 1].forEach((sign) => {
        const baselineZ = sign * zEdge;
        const laneEndZ = sign * (zEdge - laneDepth);
        const basketZ = sign * (zEdge - basketInset);

        pushLine(-laneWidth / 2, baselineZ, -laneWidth / 2, laneEndZ);
        pushLine(laneWidth / 2, baselineZ, laneWidth / 2, laneEndZ);
        pushLine(-laneWidth / 2, laneEndZ, laneWidth / 2, laneEndZ);
        pushArc(0, laneEndZ, freeThrowRadius, 0, Math.PI * 2, 32);

        if (sign > 0) {
          pushArc(0, basketZ, restrictedRadius, Math.PI, Math.PI * 2, 22);
          pushArc(0, basketZ, threePointRadius, Math.PI, Math.PI * 2, 42);
        } else {
          pushArc(0, basketZ, restrictedRadius, 0, Math.PI, 22);
          pushArc(0, basketZ, threePointRadius, 0, Math.PI, 42);
        }
      });

      if (!positions.length) return null;
      const geometry = new THREE.BufferGeometry();
      geometry.setAttribute(
        "position",
        new THREE.Float32BufferAttribute(positions, 3),
      );
      const material = new THREE.LineBasicMaterial({
        color,
        transparent: true,
        opacity,
        depthWrite: false,
      });
      const lines = new THREE.LineSegments(geometry, material);
      lines.name = `${namePrefix}-markings`;
      lines.renderOrder = 20;
      lines.frustumCulled = false;
      return lines;
    },

    addStandaloneBasketballHoopPair(
      parent,
      courtLength,
      courtWidth,
      namePrefix = "basketball-court",
    ) {
      if (!parent) return;
      const length = Math.max(6, Number(courtLength) || 0);
      const width = Math.max(4, Number(courtWidth) || 0);
      const halfLength = length / 2;
      const poleMaterial = new THREE.MeshStandardMaterial({
        color: 0x30363b,
        emissive: 0x090b0d,
        emissiveIntensity: 0.18,
        roughness: 0.48,
        metalness: 0.32,
      });
      const backboardMaterial = new THREE.MeshStandardMaterial({
        color: 0xf4f8f8,
        emissive: 0x546068,
        emissiveIntensity: 0.08,
        roughness: 0.35,
        metalness: 0.04,
        transparent: true,
        opacity: 0.9,
      });
      const rimMaterial = new THREE.MeshStandardMaterial({
        color: 0xe86224,
        emissive: 0x5f1c06,
        emissiveIntensity: 0.26,
        roughness: 0.42,
        metalness: 0.16,
      });
      const netMaterial = new THREE.MeshBasicMaterial({
        color: 0xf7ffff,
        wireframe: true,
        transparent: true,
        opacity: 0.62,
        depthWrite: false,
      });

      [-1, 1].forEach((sign) => {
        const endInset = Math.max(0.22, Math.min(0.42, length * 0.014));
        const poleZ = sign * (halfLength - endInset);
        const backboardInset = Math.max(0.9, Math.min(1.2, length * 0.043));
        const backboardZ = sign * (halfLength - backboardInset);
        const rimZ = backboardZ - sign * 0.42;
        const poleHeight = 3.5;

        const pole = new THREE.Mesh(
          new THREE.CylinderGeometry(0.09, 0.13, poleHeight, 10, 1),
          poleMaterial,
        );
        pole.name = `${namePrefix}-hoop-pole-${sign > 0 ? "a" : "b"}`;
        pole.position.set(0, poleHeight / 2, poleZ);
        pole.renderOrder = 21;
        parent.add(pole);

        const supportLength = Math.max(0.15, Math.abs(poleZ - backboardZ));
        const support = new THREE.Mesh(
          new THREE.BoxGeometry(0.11, 0.11, supportLength),
          poleMaterial,
        );
        support.name = `${namePrefix}-hoop-support-${sign > 0 ? "a" : "b"}`;
        support.position.set(0, 3.42, (poleZ + backboardZ) / 2);
        support.renderOrder = 21;
        parent.add(support);

        const backboard = new THREE.Mesh(
          new THREE.BoxGeometry(Math.min(1.83, width * 0.18), 1.05, 0.075),
          backboardMaterial,
        );
        backboard.name = `${namePrefix}-backboard-${sign > 0 ? "a" : "b"}`;
        backboard.position.set(0, 3.46, backboardZ);
        backboard.renderOrder = 22;
        parent.add(backboard);

        const rim = new THREE.Mesh(
          new THREE.TorusGeometry(0.23, 0.032, 8, 26),
          rimMaterial,
        );
        rim.name = `${namePrefix}-rim-${sign > 0 ? "a" : "b"}`;
        rim.rotation.x = Math.PI / 2;
        rim.position.set(0, 3.05, rimZ);
        rim.renderOrder = 23;
        parent.add(rim);

        const net = new THREE.Mesh(
          new THREE.CylinderGeometry(0.22, 0.14, 0.45, 12, 1, true),
          netMaterial,
        );
        net.name = `${namePrefix}-net-${sign > 0 ? "a" : "b"}`;
        net.position.set(0, 2.8, rimZ);
        net.renderOrder = 23;
        parent.add(net);
      });
    },

    createStandaloneSoccerMarkings(
      courtLength,
      courtWidth,
      {
        namePrefix = "muga-court",
        color = 0xffffff,
        opacity = 0.96,
        y = 0.07,
      } = {},
    ) {
      const length = Math.max(8, Number(courtLength) || 0);
      const width = Math.max(5, Number(courtWidth) || 0);
      const halfLength = length / 2;
      const halfWidth = width / 2;
      const positions = [];
      const pushLine = (x1, z1, x2, z2) => {
        positions.push(x1, y, z1, x2, y, z2);
      };
      const pushArc = (
        centerX,
        centerZ,
        radius,
        startAngle,
        endAngle,
        segments = 32,
      ) => {
        let previous = null;
        for (let index = 0; index <= segments; index += 1) {
          const angle = THREE.MathUtils.lerp(
            startAngle,
            endAngle,
            index / segments,
          );
          const point = {
            x: centerX + Math.cos(angle) * radius,
            z: centerZ + Math.sin(angle) * radius,
          };
          if (previous) pushLine(previous.x, previous.z, point.x, point.z);
          previous = point;
        }
      };

      const inset = Math.max(0.22, Math.min(0.48, width * 0.025));
      const xEdge = halfWidth - inset;
      const zEdge = halfLength - inset;
      pushLine(-xEdge, -zEdge, xEdge, -zEdge);
      pushLine(xEdge, -zEdge, xEdge, zEdge);
      pushLine(xEdge, zEdge, -xEdge, zEdge);
      pushLine(-xEdge, zEdge, -xEdge, -zEdge);
      pushLine(-xEdge, 0, xEdge, 0);
      pushArc(
        0,
        0,
        Math.max(1.5, Math.min(2.6, width * 0.16)),
        0,
        Math.PI * 2,
        40,
      );

      const penaltyWidth = Math.max(4.2, Math.min(9.0, width * 0.62));
      const penaltyDepth = Math.max(2.2, Math.min(4.6, length * 0.16));
      const goalAreaWidth = Math.max(2.8, Math.min(5.2, width * 0.36));
      const goalAreaDepth = Math.max(1.1, Math.min(2.1, length * 0.075));

      [-1, 1].forEach((sign) => {
        const baselineZ = sign * zEdge;
        const penaltyZ = sign * (zEdge - penaltyDepth);
        const goalAreaZ = sign * (zEdge - goalAreaDepth);

        pushLine(-penaltyWidth / 2, baselineZ, -penaltyWidth / 2, penaltyZ);
        pushLine(penaltyWidth / 2, baselineZ, penaltyWidth / 2, penaltyZ);
        pushLine(-penaltyWidth / 2, penaltyZ, penaltyWidth / 2, penaltyZ);

        pushLine(-goalAreaWidth / 2, baselineZ, -goalAreaWidth / 2, goalAreaZ);
        pushLine(goalAreaWidth / 2, baselineZ, goalAreaWidth / 2, goalAreaZ);
        pushLine(-goalAreaWidth / 2, goalAreaZ, goalAreaWidth / 2, goalAreaZ);
      });

      const geometry = new THREE.BufferGeometry();
      geometry.setAttribute(
        "position",
        new THREE.Float32BufferAttribute(positions, 3),
      );
      const material = new THREE.LineBasicMaterial({
        color,
        transparent: true,
        opacity,
        depthWrite: false,
      });
      const lines = new THREE.LineSegments(geometry, material);
      lines.name = `${namePrefix}-soccer-markings`;
      lines.renderOrder = 20;
      lines.frustumCulled = false;
      return lines;
    },

    addStandaloneMugaGoals(
      parent,
      courtLength,
      courtWidth,
      namePrefix = "muga-court",
    ) {
      if (!parent) return;
      const length = Math.max(8, Number(courtLength) || 0);
      const width = Math.max(5, Number(courtWidth) || 0);
      const halfLength = length / 2;
      const goalWidth = Math.max(2.5, Math.min(3.2, width * 0.2));
      const goalHeight = 2.0;
      const goalDepth = 0.9;
      const barThickness = 0.07;
      const frameMaterial = new THREE.MeshStandardMaterial({
        color: 0x31383c,
        emissive: 0x07090a,
        emissiveIntensity: 0.12,
        roughness: 0.45,
        metalness: 0.38,
      });
      const netMaterial = new THREE.LineBasicMaterial({
        color: 0xcfd7d8,
        transparent: true,
        opacity: 0.42,
        depthWrite: false,
      });

      [-1, 1].forEach((sign) => {
        const frontZ = sign * (halfLength - 0.18);
        const backZ = frontZ - sign * goalDepth;
        [-1, 1].forEach((xSign) => {
          const post = new THREE.Mesh(
            new THREE.BoxGeometry(barThickness, goalHeight, barThickness),
            frameMaterial,
          );
          post.position.set((xSign * goalWidth) / 2, goalHeight / 2, frontZ);
          post.name = `${namePrefix}-goal-post-${sign}-${xSign}`;
          post.renderOrder = 22;
          parent.add(post);

          const rearPost = new THREE.Mesh(
            new THREE.BoxGeometry(barThickness, goalHeight, barThickness),
            frameMaterial,
          );
          rearPost.position.set((xSign * goalWidth) / 2, goalHeight / 2, backZ);
          rearPost.name = `${namePrefix}-goal-rear-post-${sign}-${xSign}`;
          rearPost.renderOrder = 22;
          parent.add(rearPost);

          const topSide = new THREE.Mesh(
            new THREE.BoxGeometry(barThickness, barThickness, goalDepth),
            frameMaterial,
          );
          topSide.position.set(
            (xSign * goalWidth) / 2,
            goalHeight,
            (frontZ + backZ) / 2,
          );
          topSide.renderOrder = 22;
          parent.add(topSide);
        });

        const crossbar = new THREE.Mesh(
          new THREE.BoxGeometry(goalWidth, barThickness, barThickness),
          frameMaterial,
        );
        crossbar.position.set(0, goalHeight, frontZ);
        crossbar.name = `${namePrefix}-goal-crossbar-${sign}`;
        crossbar.renderOrder = 22;
        parent.add(crossbar);

        const rearCrossbar = new THREE.Mesh(
          new THREE.BoxGeometry(goalWidth, barThickness, barThickness),
          frameMaterial,
        );
        rearCrossbar.position.set(0, goalHeight, backZ);
        rearCrossbar.renderOrder = 22;
        parent.add(rearCrossbar);

        const netGeometry = new THREE.EdgesGeometry(
          new THREE.BoxGeometry(goalWidth, goalHeight, goalDepth),
        );
        const net = new THREE.LineSegments(netGeometry, netMaterial);
        net.position.set(0, goalHeight / 2, (frontZ + backZ) / 2);
        net.name = `${namePrefix}-goal-net-${sign}`;
        net.renderOrder = 21;
        parent.add(net);
      });
    },

    createStandaloneMugaFence(outer, deckTopY, namePrefix = "muga-court") {
      if (!Array.isArray(outer) || outer.length < 3) return null;
      const positions = [];
      const fenceHeight = 3.0;
      const railHeights = [0.12, 1.5, fenceHeight];

      outer.forEach((point, index) => {
        const next = outer[(index + 1) % outer.length];
        const length = point.distanceTo(next);
        const divisions = Math.max(1, Math.min(18, Math.ceil(length / 3.0)));

        for (let step = 0; step <= divisions; step += 1) {
          const t = step / divisions;
          const x = THREE.MathUtils.lerp(point.x, next.x, t);
          const z = THREE.MathUtils.lerp(point.y, next.y, t);
          positions.push(x, deckTopY + 0.04, z, x, deckTopY + fenceHeight, z);
        }

        for (let step = 0; step < divisions; step += 1) {
          const t0 = step / divisions;
          const t1 = (step + 1) / divisions;
          const x0 = THREE.MathUtils.lerp(point.x, next.x, t0);
          const z0 = THREE.MathUtils.lerp(point.y, next.y, t0);
          const x1 = THREE.MathUtils.lerp(point.x, next.x, t1);
          const z1 = THREE.MathUtils.lerp(point.y, next.y, t1);
          railHeights.forEach((height) => {
            positions.push(
              x0,
              deckTopY + height,
              z0,
              x1,
              deckTopY + height,
              z1,
            );
          });
        }
      });

      const geometry = new THREE.BufferGeometry();
      geometry.setAttribute(
        "position",
        new THREE.Float32BufferAttribute(positions, 3),
      );
      const material = new THREE.LineBasicMaterial({
        color: 0x30373a,
        transparent: true,
        opacity: 0.9,
        depthWrite: true,
      });
      const fence = new THREE.LineSegments(geometry, material);
      fence.name = `${namePrefix}-three-meter-fence`;
      fence.renderOrder = 22;
      fence.frustumCulled = false;
      return fence;
    },

    createStandaloneFullBasketballCourtGroup(
      localRings,
      modelData,
      properties = {},
    ) {
      if (!Array.isArray(localRings) || !localRings.length) return null;
      const outer = localRings[0];
      if (!Array.isArray(outer) || outer.length < 3) return null;
      const frame = this.getStandaloneRectangularCourtFrame(outer);
      if (!frame) return null;

      const group = new THREE.Group();
      group.name = "full-basketball-court";
      group.userData.appModel =
        this.getStandaloneAppModel(properties) || "basketball_court";

      const base = this.createStandaloneSportCourtBase(localRings, modelData, {
        name: "full-basketball-court",
        color: 0xbd3e3b,
        emissive: 0x3c0d0c,
        outlineColor: 0xffe8e2,
        slabLift: 0.09,
      });
      if (!base) return null;
      group.add(base.group);

      const details = new THREE.Group();
      details.name = "full-basketball-court-details";
      details.position.set(frame.center.x, base.deckTopY, frame.center.y);
      details.rotation.y = frame.rotationY;

      // Keep the painted rectangle slightly inside the real ArcGIS footprint so
      // the red slab itself remains the exact GIS polygon.
      const courtLength = Math.max(6, frame.length * 0.94);
      const courtWidth = Math.max(4, frame.width * 0.94);
      const markings = this.createStandaloneBasketballMarkings(
        courtLength,
        courtWidth,
        {
          namePrefix: "full-basketball-court",
          color: 0xffffff,
          opacity: 0.98,
          y: 0.055,
          includeBoundary: true,
        },
      );
      if (markings) details.add(markings);
      this.addStandaloneBasketballHoopPair(
        details,
        courtLength,
        courtWidth,
        "full-basketball-court",
      );

      group.add(details);
      return group;
    },

    createStandaloneMugaCourtGroup(localRings, modelData, properties = {}) {
      if (!Array.isArray(localRings) || !localRings.length) return null;
      const outer = localRings[0];
      if (!Array.isArray(outer) || outer.length < 3) return null;
      const frame = this.getStandaloneRectangularCourtFrame(outer);
      if (!frame) return null;

      const group = new THREE.Group();
      group.name = "muga-court";
      group.userData.appModel =
        this.getStandaloneAppModel(properties) || "muga_court";

      const base = this.createStandaloneSportCourtBase(localRings, modelData, {
        name: "muga-court",
        color: 0x13999a,
        emissive: 0x063f42,
        outlineColor: 0xdfffff,
        slabLift: 0.09,
      });
      if (!base) return null;
      group.add(base.group);

      const details = new THREE.Group();
      details.name = "muga-court-details";
      details.position.set(frame.center.x, base.deckTopY, frame.center.y);
      details.rotation.y = frame.rotationY;

      const courtLength = Math.max(8, frame.length * 0.94);
      const courtWidth = Math.max(5, frame.width * 0.94);

      // Soccer markings form the primary rectangular field. Basketball lines are
      // overlaid on the same surface, matching the DetailedDesign MUGA sheet.
      const soccerMarkings = this.createStandaloneSoccerMarkings(
        courtLength,
        courtWidth,
        {
          namePrefix: "muga-court",
          color: 0xf8ffff,
          opacity: 0.96,
          y: 0.065,
        },
      );
      if (soccerMarkings) details.add(soccerMarkings);

      const basketballMarkings = this.createStandaloneBasketballMarkings(
        courtLength,
        courtWidth,
        {
          namePrefix: "muga-court-basketball",
          color: 0xe5ffff,
          opacity: 0.86,
          y: 0.075,
          includeBoundary: false,
        },
      );
      if (basketballMarkings) details.add(basketballMarkings);

      this.addStandaloneBasketballHoopPair(
        details,
        courtLength,
        courtWidth,
        "muga-court",
      );
      this.addStandaloneMugaGoals(
        details,
        courtLength,
        courtWidth,
        "muga-court",
      );
      group.add(details);

      // The DetailedDesign specifies an approximately 3 m perimeter fence. Use
      // the exact GIS polygon rather than a synthetic rectangle so every MUGA
      // feature keeps its real footprint and rotation.
      const fence = this.createStandaloneMugaFence(
        outer,
        base.deckTopY,
        "muga-court",
      );
      if (fence) group.add(fence);

      return group;
    },

    getStandaloneSkateparkFrame(outer, slopeSide = "west") {
      if (!Array.isArray(outer) || outer.length < 3) return null;

      // Fit a minimum-area oriented rectangle to the actual ArcGIS footprint.
      // The previous skatepark reused the basketball PCA frame. Because this
      // footprint has an irregular western tail, PCA rotated the synthetic
      // rectangle away from the GIS cut-out. Testing every polygon-edge angle
      // gives the rectangle that actually follows the footprint on the ground.
      let best = null;
      const epsilon = 0.000001;

      for (let edgeIndex = 0; edgeIndex < outer.length; edgeIndex += 1) {
        const first = outer[edgeIndex];
        const second = outer[(edgeIndex + 1) % outer.length];
        const dx = Number(second.x) - Number(first.x);
        const dz = Number(second.y) - Number(first.y);
        const edgeLength = Math.hypot(dx, dz);
        if (edgeLength < epsilon) continue;

        // Candidate local X axis follows this footprint edge. Candidate local Z
        // axis is perpendicular. Project the whole polygon into that frame.
        const axisX = new THREE.Vector2(dx / edgeLength, dz / edgeLength);
        const axisZ = new THREE.Vector2(-axisX.y, axisX.x);
        let minX = Infinity;
        let maxX = -Infinity;
        let minZ = Infinity;
        let maxZ = -Infinity;

        outer.forEach((point) => {
          const vector = new THREE.Vector2(Number(point.x), Number(point.y));
          const px = vector.dot(axisX);
          const pz = vector.dot(axisZ);
          minX = Math.min(minX, px);
          maxX = Math.max(maxX, px);
          minZ = Math.min(minZ, pz);
          maxZ = Math.max(maxZ, pz);
        });

        const width = maxX - minX;
        const length = maxZ - minZ;
        const area = width * length;
        if (!best || area < best.area) {
          best = { axisX, axisZ, minX, maxX, minZ, maxZ, width, length, area };
        }
      }

      if (!best) return null;

      // Either fitted rectangle axis may be the ramp direction. Pick the axis
      // whose end points most strongly toward APP_HOOP_SIDE, then orient its
      // positive direction toward that side. This preserves the footprint
      // rotation while still making APP_HOOP_SIDE=west control the ramp end.
      const targets = {
        north: new THREE.Vector2(0, -1),
        east: new THREE.Vector2(1, 0),
        south: new THREE.Vector2(0, 1),
        west: new THREE.Vector2(-1, 0),
      };
      const target = targets[slopeSide] || targets.west;

      let forward;
      let widthAxis;
      let length;
      let width;
      let centerForward;
      let centerWidth;

      if (
        Math.abs(best.axisZ.dot(target)) >= Math.abs(best.axisX.dot(target))
      ) {
        forward = best.axisZ.clone();
        widthAxis = best.axisX.clone();
        length = best.length;
        width = best.width;
        centerForward = (best.minZ + best.maxZ) / 2;
        centerWidth = (best.minX + best.maxX) / 2;
      } else {
        forward = best.axisX.clone();
        widthAxis = best.axisZ.clone();
        length = best.width;
        width = best.length;
        centerForward = (best.minX + best.maxX) / 2;
        centerWidth = (best.minZ + best.maxZ) / 2;
      }

      if (forward.dot(target) < 0) forward.multiplyScalar(-1);

      // Rebuild widthAxis from forward so it exactly matches Three.js local +X
      // after rotation. Re-project only for the center; dimensions remain those
      // of the minimum-area rectangle.
      widthAxis = new THREE.Vector2(forward.y, -forward.x).normalize();
      let minForward = Infinity;
      let maxForward = -Infinity;
      let minWidth = Infinity;
      let maxWidth = -Infinity;
      outer.forEach((point) => {
        const vector = new THREE.Vector2(Number(point.x), Number(point.y));
        const pf = vector.dot(forward);
        const pw = vector.dot(widthAxis);
        minForward = Math.min(minForward, pf);
        maxForward = Math.max(maxForward, pf);
        minWidth = Math.min(minWidth, pw);
        maxWidth = Math.max(maxWidth, pw);
      });

      centerForward = (minForward + maxForward) / 2;
      centerWidth = (minWidth + maxWidth) / 2;
      const center = forward
        .clone()
        .multiplyScalar(centerForward)
        .add(widthAxis.clone().multiplyScalar(centerWidth));

      return {
        center,
        forward,
        widthAxis,
        rotationY: Math.atan2(forward.x, forward.y),
        length: Math.max(2, maxForward - minForward),
        width: Math.max(2, maxWidth - minWidth),
      };
    },

    createStandaloneSkateparkGroup(localRings, modelData, properties = {}) {
      if (!Array.isArray(localRings) || !localRings.length) return null;
      const outer = localRings[0];
      if (!Array.isArray(outer) || outer.length < 3) return null;

      const slopeSide = this.getStandaloneSkateparkSlopeSide(properties);
      const frame = this.getStandaloneSkateparkFrame(outer, slopeSide);
      if (!frame) return null;

      const group = new THREE.Group();
      group.name = "skatepark";
      group.userData.appModel = "skatepark";
      group.userData.slopeSide = slopeSide;

      const terrainSupportY = this.getStandalonePolygonSupportHeight(
        localRings,
        modelData,
      );
      const smallerDimension = Math.max(4, Math.min(frame.width, frame.length));
      const bowlDepth = THREE.MathUtils.clamp(
        smallerDimension * 0.11,
        1.25,
        1.65,
      );

      // Keep the bowl floor just above Terrain (After) so the terrain can never
      // visually cut through the concrete riding surface.
      const bowlFloorY = terrainSupportY + 0.12;
      const deckTopY = bowlFloorY + bowlDepth;

      const details = new THREE.Group();
      details.name = "skatepark-details";
      details.position.set(frame.center.x, deckTopY, frame.center.y);
      details.rotation.y = frame.rotationY;

      const localToWorld = (x, z) =>
        frame.center
          .clone()
          .add(frame.widthAxis.clone().multiplyScalar(x))
          .add(frame.forward.clone().multiplyScalar(z));

      const groundRelativeY = (x, z) => {
        const world = localToWorld(x, z);
        return (
          this.getStandaloneTerrainHeightAtPoint(
            { x: world.x, z: world.y },
            modelData,
          ) +
          0.045 -
          deckTopY
        );
      };

      const concreteMaterial = new THREE.MeshStandardMaterial({
        color: 0xb9c0c3,
        emissive: 0x202628,
        emissiveIntensity: 0.045,
        roughness: 0.94,
        metalness: 0.01,
        side: THREE.DoubleSide,
        depthWrite: true,
        polygonOffset: true,
        polygonOffsetFactor: -4,
        polygonOffsetUnits: -8,
      });

      // Separate material for exposed concrete edges. The previous mesh reused the
      // top material on very steep / vertical faces; in the park lighting those
      // faces could become almost black. Give retaining and ramp side faces a
      // slightly brighter concrete response so they read as solid concrete.
      const concreteSideMaterial = new THREE.MeshStandardMaterial({
        color: 0xaeb7ba,
        emissive: 0x3b4346,
        emissiveIntensity: 0.12,
        roughness: 0.96,
        metalness: 0.0,
        side: THREE.DoubleSide,
        depthTest: true,
        depthWrite: true,
        polygonOffset: true,
        polygonOffsetFactor: -3,
        polygonOffsetUnits: -6,
      });

      // Use the REAL ArcGIS footprint as the outer concrete outline.
      const outerLocal = outer.map((point) => {
        const offset = new THREE.Vector2(
          Number(point.x) - frame.center.x,
          Number(point.y) - frame.center.y,
        );
        return new THREE.Vector2(
          offset.dot(frame.widthAxis),
          offset.dot(frame.forward),
        );
      });
      if (
        outerLocal.length > 3 &&
        outerLocal[0].distanceToSquared(outerLocal[outerLocal.length - 1]) <
          0.000001
      ) {
        outerLocal.pop();
      }

      const eastZ = Math.min(...outerLocal.map((point) => point.y));
      const westZ = Math.max(...outerLocal.map((point) => point.y));
      const footprintLength = Math.max(2, westZ - eastZ);

      // The western extension is the roll-in/ramp. Local +Z points toward the
      // APP_HOOP_SIDE value, so APP_HOOP_SIDE=west identifies the correct end.
      const rampLength = THREE.MathUtils.clamp(
        footprintLength * 0.31,
        3.0,
        6.5,
      );
      const rampStartZ = westZ - rampLength;

      const clipPolygonAtZ = (points, boundaryZ, keepLessEqual) => {
        if (!Array.isArray(points) || points.length < 3) return [];
        const result = [];
        const inside = (point) =>
          keepLessEqual
            ? point.y <= boundaryZ + 0.000001
            : point.y >= boundaryZ - 0.000001;

        for (let index = 0; index < points.length; index += 1) {
          const current = points[index];
          const previous = points[(index + points.length - 1) % points.length];
          const currentInside = inside(current);
          const previousInside = inside(previous);

          if (currentInside !== previousInside) {
            const dz = current.y - previous.y;
            const t =
              Math.abs(dz) > 0.000001 ? (boundaryZ - previous.y) / dz : 0;
            result.push(
              new THREE.Vector2(
                THREE.MathUtils.lerp(previous.x, current.x, t),
                boundaryZ,
              ),
            );
          }
          if (currentInside) result.push(current.clone());
        }
        return result;
      };

      const deckOuter = clipPolygonAtZ(outerLocal, rampStartZ, true);
      const rampOuter = clipPolygonAtZ(outerLocal, rampStartZ, false);
      if (deckOuter.length < 3 || rampOuter.length < 3) return null;

      // Smooth a polygon while keeping the resulting ring safely inside it.
      const chaikin = (points, iterations = 2) => {
        let result = points.map((point) => point.clone());
        for (let iteration = 0; iteration < iterations; iteration += 1) {
          const next = [];
          for (let index = 0; index < result.length; index += 1) {
            const a = result[index];
            const b = result[(index + 1) % result.length];
            next.push(
              new THREE.Vector2(
                THREE.MathUtils.lerp(a.x, b.x, 0.25),
                THREE.MathUtils.lerp(a.y, b.y, 0.25),
              ),
            );
            next.push(
              new THREE.Vector2(
                THREE.MathUtils.lerp(a.x, b.x, 0.75),
                THREE.MathUtils.lerp(a.y, b.y, 0.75),
              ),
            );
          }
          result = next;
        }
        return result;
      };

      const deckCentroid = deckOuter
        .reduce((sum, point) => sum.add(point), new THREE.Vector2())
        .multiplyScalar(1 / deckOuter.length);

      // The bowl opening now follows the SAME plan shape as the raised bowl block
      // (ignoring the west ramp) instead of forcing a generic rounded rectangle.
      // Shrinking toward the deck centroid makes an inset copy; Chaikin smoothing
      // turns the GIS corners into realistic continuous skate-bowl curves.
      const deckRingForPointTest = deckOuter.map((point) => ({
        x: point.x,
        y: point.y,
      }));
      let openingRing = null;
      for (let scale = 0.72; scale >= 0.52; scale -= 0.025) {
        const inset = deckOuter.map((point) =>
          deckCentroid
            .clone()
            .add(point.clone().sub(deckCentroid).multiplyScalar(scale)),
        );
        const candidate = chaikin(inset, 2);
        const fits = candidate.every((point) =>
          this.isStandalonePointInsidePolygonRing(
            { x: point.x, y: point.y },
            deckRingForPointTest,
          ),
        );
        if (fits) {
          openingRing = candidate;
          break;
        }
      }
      if (!openingRing) {
        openingRing = chaikin(
          deckOuter.map((point) =>
            deckCentroid
              .clone()
              .add(point.clone().sub(deckCentroid).multiplyScalar(0.5)),
          ),
          2,
        );
      }

      // --- GIS-FITTED BOWL DECK --------------------------------------------
      const deckFaces = THREE.ShapeUtils.triangulateShape(deckOuter, [
        openingRing,
      ]);
      const deckFlat = deckOuter.concat(openingRing);
      const deckPositions = [];
      deckFaces.forEach((face) => {
        face.forEach((index) => {
          const point = deckFlat[index];
          deckPositions.push(point.x, 0, point.y);
        });
      });

      if (deckPositions.length) {
        const geometry = new THREE.BufferGeometry();
        geometry.setAttribute(
          "position",
          new THREE.Float32BufferAttribute(deckPositions, 3),
        );
        geometry.computeVertexNormals();
        geometry.computeBoundingSphere();
        const mesh = new THREE.Mesh(geometry, concreteMaterial);
        mesh.name = "skatepark-gis-fitted-bowl-deck";
        mesh.renderOrder = 14;
        mesh.frustumCulled = false;
        details.add(mesh);
      }

      // --- WEST RAMP / APRON -----------------------------------------------
      // One continuous surface: the seam touching the bowl deck stays at deck
      // height, while every true western perimeter point drops to Terrain (After).
      // There is NO vertical face at the seam, so the ramp is not separated from
      // the bowl by a concrete wall.
      const rampFaces = THREE.ShapeUtils.triangulateShape(rampOuter, []);
      const rampPositions = [];
      const rampVertexY = (point) => {
        // Continuous west roll-in: the seam shared with the bowl deck is exactly
        // deck height, and the apron then descends smoothly toward Terrain (After).
        // Using one height function across the whole ramp prevents the triangulated
        // ramp from folding into a near-vertical face that looked like a wall
        // between the ramp and the bowl.
        const span = Math.max(0.001, westZ - rampStartZ);
        const progress = THREE.MathUtils.clamp(
          (point.y - rampStartZ) / span,
          0,
          1,
        );
        const eased = progress * progress * (3 - 2 * progress);
        const groundY = groundRelativeY(point.x, point.y);
        return THREE.MathUtils.lerp(0, groundY, eased);
      };

      rampFaces.forEach((face) => {
        face.forEach((index) => {
          const point = rampOuter[index];
          rampPositions.push(point.x, rampVertexY(point), point.y);
        });
      });

      if (rampPositions.length) {
        const geometry = new THREE.BufferGeometry();
        geometry.setAttribute(
          "position",
          new THREE.Float32BufferAttribute(rampPositions, 3),
        );
        geometry.computeVertexNormals();
        geometry.computeBoundingSphere();
        const ramp = new THREE.Mesh(geometry, concreteMaterial);
        ramp.name = "skatepark-gis-fitted-west-ground-ramp";
        ramp.renderOrder = 15;
        ramp.frustumCulled = false;
        details.add(ramp);
      }

      // --- CLOSED OUTER CONCRETE SIDES --------------------------------------
      // Close the object against Terrain (After). There are two different cases:
      //   1) raised bowl-block perimeter -> visible concrete retaining wall
      //   2) west ramp perimeter        -> NO wall; ramp surface itself meets grade
      //
      // Crucially, we do NOT create a face on the artificial rampStartZ seam, so
      // there is still no wall separating the ramp from the bowl/deck. Splitting
      // original ArcGIS perimeter edges at rampStartZ also fixes the old missing
      // triangular faces that appeared black.
      const raisedSidePositions = [];
      const rampSidePositions = [];

      const intersectionAtRampStart = (a, b) => {
        const dz = b.y - a.y;
        if (Math.abs(dz) < 0.000001) return a.clone();
        const t = THREE.MathUtils.clamp((rampStartZ - a.y) / dz, 0, 1);
        return new THREE.Vector2(THREE.MathUtils.lerp(a.x, b.x, t), rampStartZ);
      };

      const appendSideQuad = (positions, a, b, topYForPoint) => {
        if (!a || !b || a.distanceToSquared(b) < 0.0000001) return;
        const topA = topYForPoint(a);
        const topB = topYForPoint(b);
        const groundA = groundRelativeY(a.x, a.y);
        const groundB = groundRelativeY(b.x, b.y);

        // If this edge is already essentially touching grade there is no visible
        // wall to add. Otherwise close it with two triangles.
        if (
          Math.abs(topA - groundA) < 0.006 &&
          Math.abs(topB - groundB) < 0.006
        ) {
          return;
        }

        positions.push(
          a.x,
          topA,
          a.y,
          a.x,
          groundA,
          a.y,
          b.x,
          groundB,
          b.y,
          a.x,
          topA,
          a.y,
          b.x,
          groundB,
          b.y,
          b.x,
          topB,
          b.y,
        );
      };

      for (let index = 0; index < outerLocal.length; index += 1) {
        const first = outerLocal[index];
        const second = outerLocal[(index + 1) % outerLocal.length];
        const firstRaised = first.y <= rampStartZ + 0.000001;
        const secondRaised = second.y <= rampStartZ + 0.000001;

        // Only the true raised bowl block gets a retaining wall. Any perimeter
        // segment that reaches the ramp transition is intentionally left open so
        // the concrete roll-in can blend directly into grade with no vertical
        // divider/wedge beside the bowl.
        const rampBlendMargin = Math.max(0.35, rampLength * 0.12);
        if (
          firstRaised &&
          secondRaised &&
          first.y < rampStartZ - rampBlendMargin &&
          second.y < rampStartZ - rampBlendMargin
        ) {
          appendSideQuad(raisedSidePositions, first, second, () => 0);
        }
      }

      const addConcreteSideMesh = (positions, name, order) => {
        if (!positions.length) return;
        const geometry = new THREE.BufferGeometry();
        geometry.setAttribute(
          "position",
          new THREE.Float32BufferAttribute(positions, 3),
        );
        geometry.computeVertexNormals();
        geometry.computeBoundingSphere();
        const mesh = new THREE.Mesh(geometry, concreteSideMaterial);
        mesh.name = name;
        mesh.renderOrder = order;
        mesh.frustumCulled = false;
        details.add(mesh);
      };

      addConcreteSideMesh(
        raisedSidePositions,
        "skatepark-bowl-block-concrete-retaining-walls",
        13,
      );
      // No side mesh is added for the west ramp. Its outer boundary vertices
      // are already sampled at Terrain (After), so the concrete apron visually
      // merges into the ground with no wall along the blue-circled side.

      // --- BOWL -------------------------------------------------------------
      // The bowl is a family of inset copies of the SAME opening shape. This makes
      // the inside walls match the outside bowl block rather than reverting to a
      // capsule/rounded rectangle.
      const bowlCenter = openingRing
        .reduce((sum, point) => sum.add(point), new THREE.Vector2())
        .multiplyScalar(1 / openingRing.length);
      const innerScale = 0.58;
      const floorRing = openingRing.map((point) =>
        bowlCenter
          .clone()
          .add(point.clone().sub(bowlCenter).multiplyScalar(innerScale)),
      );
      const ringCount = 18;
      const angularSegments = openingRing.length;
      const bowlPositions = [];
      const bowlColours = [];
      const bowlIndices = [];
      const lipColour = new THREE.Color(0xcbd0d2);
      const floorColour = new THREE.Color(0x8e979b);

      const ringAt = (t) => {
        const eased = t * t * (3 - 2 * t);
        return openingRing.map((outerPoint, index) => {
          const innerPoint = floorRing[index];
          return new THREE.Vector2(
            THREE.MathUtils.lerp(innerPoint.x, outerPoint.x, eased),
            THREE.MathUtils.lerp(innerPoint.y, outerPoint.y, eased),
          );
        });
      };

      for (let ringIndex = 0; ringIndex < ringCount; ringIndex += 1) {
        const t = ringIndex / (ringCount - 1);
        const y = -bowlDepth + bowlDepth * (t * t * (3 - 2 * t));
        const ring = ringAt(t);
        ring.forEach((point) => {
          bowlPositions.push(point.x, y, point.y);
          const colour = floorColour.clone().lerp(lipColour, t);
          bowlColours.push(colour.r, colour.g, colour.b);
        });
      }

      for (let ringIndex = 0; ringIndex < ringCount - 1; ringIndex += 1) {
        const firstRing = ringIndex * angularSegments;
        const secondRing = (ringIndex + 1) * angularSegments;
        for (let segment = 0; segment < angularSegments; segment += 1) {
          const next = (segment + 1) % angularSegments;
          const a = firstRing + segment;
          const b = firstRing + next;
          const c = secondRing + segment;
          const d = secondRing + next;
          bowlIndices.push(a, b, c, b, d, c);
        }
      }

      // Solid riding floor: never expose Terrain below the bowl.
      const floorFaces = THREE.ShapeUtils.triangulateShape(floorRing, []);
      const floorBaseIndex = bowlPositions.length / 3;
      floorRing.forEach((point) => {
        bowlPositions.push(point.x, -bowlDepth, point.y);
        bowlColours.push(floorColour.r, floorColour.g, floorColour.b);
      });
      floorFaces.forEach((face) => {
        bowlIndices.push(
          floorBaseIndex + face[0],
          floorBaseIndex + face[1],
          floorBaseIndex + face[2],
        );
      });

      const bowlGeometry = new THREE.BufferGeometry();
      bowlGeometry.setAttribute(
        "position",
        new THREE.Float32BufferAttribute(bowlPositions, 3),
      );
      bowlGeometry.setAttribute(
        "color",
        new THREE.Float32BufferAttribute(bowlColours, 3),
      );
      bowlGeometry.setIndex(bowlIndices);
      bowlGeometry.computeVertexNormals();
      bowlGeometry.computeBoundingSphere();

      const bowl = new THREE.Mesh(
        bowlGeometry,
        new THREE.MeshStandardMaterial({
          vertexColors: true,
          roughness: 0.92,
          metalness: 0.01,
          side: THREE.DoubleSide,
          depthTest: true,
          depthWrite: true,
          polygonOffset: true,
          polygonOffsetFactor: -5,
          polygonOffsetUnits: -10,
        }),
      );
      bowl.name = "skatepark-bowl";
      bowl.renderOrder = 16;
      bowl.frustumCulled = false;
      details.add(bowl);

      const copingPoints = openingRing.map(
        (point) => new THREE.Vector3(point.x, 0.035, point.y),
      );
      const coping = new THREE.LineLoop(
        new THREE.BufferGeometry().setFromPoints(copingPoints),
        new THREE.LineBasicMaterial({
          color: 0xe5eaeb,
          transparent: true,
          opacity: 0.78,
          depthWrite: false,
        }),
      );
      coping.name = "skatepark-coping-outline";
      coping.renderOrder = 19;
      details.add(coping);

      const contourMaterial = new THREE.LineBasicMaterial({
        color: 0x586367,
        transparent: true,
        opacity: 0.18,
        depthWrite: false,
      });
      [0.35, 0.62, 0.84].forEach((t, index) => {
        const y = -bowlDepth + bowlDepth * (t * t * (3 - 2 * t)) + 0.018;
        const ring = ringAt(t);
        const points = ring.map(
          (point) => new THREE.Vector3(point.x, y, point.y),
        );
        const line = new THREE.LineLoop(
          new THREE.BufferGeometry().setFromPoints(points),
          contourMaterial,
        );
        line.name = `skatepark-bowl-contour-${index + 1}`;
        line.renderOrder = 18;
        details.add(line);
      });

      group.add(details);
      return group;
    },

    createStandaloneMergedPolygonObjects(
      layerKey,
      records,
      modelData,
      materials,
      pointRecords = [],
      contextRecords = records,
      contextPointRecords = pointRecords,
    ) {
      const group = new THREE.Group();
      const positions = [];
      const linePositions = [];
      const pergolaPositions = [];
      const pergolaLinePositions = [];
      const shrubRecords =
        layerKey === "vegetation"
          ? records.filter(({ feature, featureType }) =>
              this.isStandaloneShrubFeature(
                featureType,
                (feature && feature.properties) || {},
              ),
            )
          : [];
      const surfaceRecords = shrubRecords.length
        ? records.filter(
            ({ feature, featureType }) =>
              !this.isStandaloneShrubFeature(
                featureType,
                (feature && feature.properties) || {},
              ),
          )
        : records;
      let polygonCount = 0;
      let halfBasketballCourtCount = 0;
      let fullBasketballCourtCount = 0;
      let mugaCourtCount = 0;
      let skateparkCount = 0;
      let detailedModularBuildingCount = 0;

      const structureFootprints = [];
      const lightPoleRecords = [];
      if (layerKey === "structures") {
        contextPointRecords.forEach(({ feature, featureType }) => {
          const geometry = feature && feature.geometry;
          const properties = (feature && feature.properties) || {};
          const type = String(featureType || "").toLowerCase();
          const description = `${type} ${
            properties.Remarks || properties.QC_REMARKS || properties.Type || ""
          }`.toLowerCase();
          if (!/pole|light|lighting|fixture|lamp|luminaire/.test(description)) {
            return;
          }

          const coordinates =
            geometry && geometry.type === "Point"
              ? [geometry.coordinates]
              : geometry && geometry.type === "MultiPoint"
                ? geometry.coordinates
                : [];
          const remarks = String(
            properties.Remarks ||
              properties.QC_REMARKS ||
              properties.Type ||
              "",
          );
          const remarkHeight = remarks.match(
            /(?:^|\D)(\d+(?:\.\d+)?)\s*(?:M|$)/i,
          );
          const poleHeight = Math.max(
            2,
            Math.min(
              14,
              this.getStandaloneNumericProperty(
                properties,
                ["PoleHeight", "Height", "height_m"],
                remarkHeight ? Number(remarkHeight[1]) : 5.6,
              ),
            ),
          );

          coordinates.forEach((coordinate) => {
            if (!Array.isArray(coordinate) || coordinate.length < 2) return;
            lightPoleRecords.push({
              local: this.lngLatToTerrainMeters(coordinate, modelData.origin),
              height: poleHeight,
            });
          });
        });

        contextRecords.forEach(({ feature, featureType }) => {
          const geometry = feature && feature.geometry;
          const properties = (feature && feature.properties) || {};
          const isPergola = this.isStandaloneCoveredPergolaStructure(
            featureType,
            properties,
          );
          const height = this.getStandaloneStructureHeight(
            featureType,
            properties,
          );

          this.getStandalonePolygonParts(geometry).forEach((rings) => {
            const localRings = this.getStandaloneLocalPolygonRings(
              rings,
              modelData.origin,
            );
            if (!localRings.length) return;
            structureFootprints.push({
              bounds: this.getStandalonePolygonBounds(localRings[0]),
              height,
              isPergola,
            });
          });
        });
      }

      surfaceRecords.forEach(({ feature, featureType }) => {
        const geometry = feature && feature.geometry;
        const properties = (feature && feature.properties) || {};
        this.getStandalonePolygonParts(geometry).forEach((rings) => {
          const localRings = this.getStandaloneLocalPolygonRings(
            rings,
            modelData.origin,
          );
          if (!localRings.length) return;

          if (layerKey === "structures") {
            const structureHeight = this.getStandaloneStructureHeight(
              featureType,
              properties,
            );
            const architecturalBuildingStyle =
              this.getStandaloneArchitecturalBuildingStyle(
                featureType,
                properties,
              );
            const detailedModularStyle =
              this.getStandaloneDetailedModularBuildingStyle(
                featureType,
                properties,
              );

            const detailedEquipmentStyle =
              this.getStandaloneDetailedEquipmentStyle(featureType, properties);

            if (architecturalBuildingStyle) {
              const architecturalBuildingGroup =
                this.createStandaloneArchitecturalBuildingGroup(
                  localRings,
                  modelData,
                  architecturalBuildingStyle,
                  properties,
                );
              if (architecturalBuildingGroup) {
                group.add(architecturalBuildingGroup);
              } else {
                this.appendStandaloneExtrudedPolygonGeometry(
                  localRings,
                  modelData,
                  structureHeight,
                  positions,
                  linePositions,
                );
              }
            } else if (detailedEquipmentStyle) {
              const equipmentGroup =
                this.createStandaloneDetailedEquipmentGroup(
                  localRings,
                  modelData,
                  detailedEquipmentStyle,
                  properties,
                );
              if (equipmentGroup) {
                group.add(equipmentGroup);
              } else {
                this.appendStandaloneExtrudedPolygonGeometry(
                  localRings,
                  modelData,
                  Math.max(0.12, structureHeight),
                  positions,
                  linePositions,
                );
              }
            } else if (detailedModularStyle) {
              const modularBuildingGroup =
                this.createStandaloneDetailedModularBuildingGroup(
                  localRings,
                  modelData,
                  detailedModularStyle,
                  properties,
                );
              if (modularBuildingGroup) {
                group.add(modularBuildingGroup);
                detailedModularBuildingCount += 1;
              } else {
                this.appendStandaloneExtrudedPolygonGeometry(
                  localRings,
                  modelData,
                  structureHeight,
                  positions,
                  linePositions,
                );
              }
            } else if (this.isStandaloneSkateparkStructure(properties)) {
              const skateparkGroup = this.createStandaloneSkateparkGroup(
                localRings,
                modelData,
                properties,
              );
              if (skateparkGroup) {
                group.add(skateparkGroup);
                skateparkCount += 1;
              } else {
                this.appendStandaloneExtrudedPolygonGeometry(
                  localRings,
                  modelData,
                  0.22,
                  positions,
                  linePositions,
                );
              }
            } else if (this.isStandaloneMugaCourtStructure(properties)) {
              const mugaGroup = this.createStandaloneMugaCourtGroup(
                localRings,
                modelData,
                properties,
              );
              if (mugaGroup) {
                group.add(mugaGroup);
                mugaCourtCount += 1;
              } else {
                this.appendStandaloneExtrudedPolygonGeometry(
                  localRings,
                  modelData,
                  0.12,
                  positions,
                  linePositions,
                );
              }
            } else if (
              this.isStandaloneFullBasketballCourtStructure(properties)
            ) {
              const fullCourtGroup =
                this.createStandaloneFullBasketballCourtGroup(
                  localRings,
                  modelData,
                  properties,
                );
              if (fullCourtGroup) {
                group.add(fullCourtGroup);
                fullBasketballCourtCount += 1;
              } else {
                this.appendStandaloneExtrudedPolygonGeometry(
                  localRings,
                  modelData,
                  0.12,
                  positions,
                  linePositions,
                );
              }
            } else if (
              this.isStandaloneHalfBasketballCourtStructure(properties)
            ) {
              const courtGroup = this.createStandaloneHalfBasketballCourtGroup(
                localRings,
                modelData,
                properties,
              );
              if (courtGroup) {
                group.add(courtGroup);
                halfBasketballCourtCount += 1;
              } else {
                this.appendStandaloneExtrudedPolygonGeometry(
                  localRings,
                  modelData,
                  0.22,
                  positions,
                  linePositions,
                );
              }
            } else if (
              this.isStandaloneCoveredPergolaStructure(featureType, properties)
            ) {
              const pergolaClearance =
                this.getStandalonePergolaRequiredClearance(
                  localRings,
                  structureFootprints,
                  lightPoleRecords,
                );
              this.appendStandalonePergolaCanopyGeometry(
                localRings,
                modelData,
                Math.max(structureHeight, pergolaClearance),
                pergolaPositions,
                pergolaLinePositions,
                properties,
              );
            } else {
              this.appendStandaloneExtrudedPolygonGeometry(
                localRings,
                modelData,
                structureHeight,
                positions,
                linePositions,
              );
            }
          } else {
            // Paths are on-grade and are densely draped over Terrain (after),
            // while water keeps the highest visible surface priority.
            const vegetationDescriptor =
              layerKey === "vegetation"
                ? this.getStandaloneVegetationDescriptor(
                    featureType,
                    properties,
                  )
                : "";
            const isGroundCover =
              layerKey === "vegetation" &&
              /ground[_ -]?cover|groundcover|lawn|turf|grass/.test(
                vegetationDescriptor,
              );

            let offset =
              layerKey === "water"
                ? 0.88
                : layerKey === "paths"
                  ? THREE_PATH_TERRAIN_CLEARANCE_METERS
                  : isGroundCover
                    ? THREE_GROUNDCOVER_TERRAIN_CLEARANCE_METERS
                    : 0.28;

            if (
              layerKey === "vegetation" &&
              !isGroundCover &&
              /shrub/.test(vegetationDescriptor)
            ) {
              offset = 0.3;
            }

            const drapeGroundSurface = layerKey === "paths" || isGroundCover;

            this.appendStandaloneFlatPolygonGeometry(
              localRings,
              modelData,
              offset,
              positions,
              linePositions,
              {
                // GroundCover no longer uses the old face-lift fallback. That
                // fallback raised whole grass polygons above sloped terrain and
                // made them appear to float. Instead, grass is tessellated and
                // sampled directly against Terrain (after), like an on-grade
                // surface.
                preventTerrainOcclusion:
                  layerKey === "water" ||
                  (layerKey === "vegetation" && !isGroundCover),
                drapeToTerrain: drapeGroundSurface,
                drapeSegmentLength: isGroundCover
                  ? THREE_GROUNDCOVER_DRAPE_SEGMENT_METERS
                  : THREE_PATH_DRAPE_SEGMENT_METERS,
                maxDrapeSubdivisions: isGroundCover
                  ? THREE_GROUNDCOVER_DRAPE_MAX_SUBDIVISIONS
                  : THREE_PATH_DRAPE_MAX_SUBDIVISIONS,
                terrainClearance:
                  layerKey === "paths" ? 0.02 : isGroundCover ? 0 : 0.12,
              },
            );
          }
          polygonCount += 1;
        });
      });

      if (shrubRecords.length) {
        const shrubGroup = this.createStandaloneShrubInstanceObjects(
          shrubRecords,
          modelData,
          materials,
        );
        polygonCount += shrubGroup.userData.shrubCount || 0;
        group.add(shrubGroup);
      }

      if (positions.length) {
        const geometry = new THREE.BufferGeometry();
        geometry.setAttribute(
          "position",
          new THREE.Float32BufferAttribute(positions, 3),
        );
        geometry.computeVertexNormals();
        geometry.computeBoundingSphere();
        const mesh = new THREE.Mesh(geometry, materials.solid);
        mesh.name = `${layerKey}-polygon-surfaces`;
        mesh.frustumCulled = false;
        mesh.renderOrder =
          layerKey === "water" ? 60 : layerKey === "paths" ? 6 : 10;
        group.add(mesh);

        // PATHS use a regular square grid built into materials.solid.
        // Do not add THREE.WireframeGeometry here: that exposes triangulation
        // diagonals rather than the clean elevation grid used in the concept.
      }

      if (linePositions.length) {
        const geometry = new THREE.BufferGeometry();
        geometry.setAttribute(
          "position",
          new THREE.Float32BufferAttribute(linePositions, 3),
        );
        const lines = new THREE.LineSegments(geometry, materials.lines);
        lines.name = `${layerKey}-polygon-outlines`;
        lines.frustumCulled = false;
        lines.renderOrder =
          layerKey === "water" ? 61 : layerKey === "paths" ? 7 : 8;
        group.add(lines);
      }

      if (pergolaPositions.length || pergolaLinePositions.length) {
        const pergolaMaterials = this.createStandalonePergolaMaterials();

        if (pergolaPositions.length) {
          const geometry = new THREE.BufferGeometry();
          geometry.setAttribute(
            "position",
            new THREE.Float32BufferAttribute(pergolaPositions, 3),
          );
          geometry.computeVertexNormals();
          geometry.computeBoundingSphere();
          const mesh = new THREE.Mesh(geometry, pergolaMaterials.solid);
          mesh.name = "covered-pergola-canopy-surfaces";
          mesh.frustumCulled = false;
          mesh.renderOrder = 11;
          group.add(mesh);
        } else {
          pergolaMaterials.solid.dispose();
        }

        if (pergolaLinePositions.length) {
          const geometry = new THREE.BufferGeometry();
          geometry.setAttribute(
            "position",
            new THREE.Float32BufferAttribute(pergolaLinePositions, 3),
          );
          const lines = new THREE.LineSegments(
            geometry,
            pergolaMaterials.lines,
          );
          lines.name = "covered-pergola-canopy-outlines";
          lines.frustumCulled = false;
          lines.renderOrder = 12;
          group.add(lines);
        } else {
          pergolaMaterials.lines.dispose();
        }
      }

      group.userData.polygonCount = polygonCount;
      group.userData.halfBasketballCourtCount = halfBasketballCourtCount;
      group.userData.fullBasketballCourtCount = fullBasketballCourtCount;
      group.userData.mugaCourtCount = mugaCourtCount;
      group.userData.skateparkCount = skateparkCount;
      group.userData.detailedModularBuildingCount =
        detailedModularBuildingCount;
      return group;
    },

    createStandalonePointInstanceObjects(
      records,
      modelData,
      materials,
      layerKey = "",
    ) {
      const group = new THREE.Group();
      const poleRecords = [];
      const manholeRecords = [];
      const markerRecords = [];

      records.forEach(({ feature, featureType }) => {
        const geometry = feature && feature.geometry;
        const properties = (feature && feature.properties) || {};
        const coordinates =
          geometry.type === "Point"
            ? [geometry.coordinates]
            : geometry.type === "MultiPoint"
              ? geometry.coordinates
              : [];

        coordinates.forEach((coordinate) => {
          if (!Array.isArray(coordinate) || coordinate.length < 2) return;
          const local = this.lngLatToTerrainMeters(
            coordinate,
            modelData.origin,
          );
          const type = String(featureType || "").toLowerCase();
          const utilityLabel = this.getStandaloneUtilitySubLayerLabel(feature);
          const record = { local, properties, type, featureType, utilityLabel };
          if (utilityLabel === "Electric Manhole" || /manhole/.test(type)) {
            manholeRecords.push(record);
          } else if (
            utilityLabel === "Electric Pole" ||
            utilityLabel === "Lighting" ||
            /pole|light/.test(type)
          ) {
            poleRecords.push(record);
          } else {
            markerRecords.push(record);
          }
        });
      });

      const dummy = new THREE.Object3D();
      const up = new THREE.Vector3(0, 1, 0);

      if (poleRecords.length) {
        const geometry = new THREE.CylinderGeometry(1, 1.08, 1, 7, 1);
        const poles = new THREE.InstancedMesh(
          geometry,
          materials.solid,
          poleRecords.length,
        );
        const outlines = new THREE.InstancedMesh(
          geometry,
          materials.wire,
          poleRecords.length,
        );
        const lightPoleTransforms = [];

        poleRecords.forEach((record, index) => {
          const remarks = String(
            record.properties.Remarks ||
              record.properties.QC_REMARKS ||
              record.properties.Type ||
              "",
          );
          const remarkHeight = remarks.match(
            /(?:^|\D)(\d+(?:\.\d+)?)\s*(?:M|$)/i,
          );
          const height = Math.max(
            2,
            Math.min(
              14,
              this.getStandaloneNumericProperty(
                record.properties,
                ["PoleHeight", "Height", "height_m"],
                remarkHeight ? Number(remarkHeight[1]) : 5.6,
              ),
            ),
          );
          const baseY =
            this.getStandaloneTerrainSupportHeightAtPoint(
              record.local,
              modelData,
              0.22,
            ) + 0.1;

          dummy.position.set(
            record.local.x,
            baseY + height / 2,
            record.local.z,
          );
          dummy.quaternion.setFromUnitVectors(up, up);
          dummy.scale.set(0.13, height, 0.13);
          dummy.updateMatrix();
          poles.setMatrixAt(index, dummy.matrix);

          dummy.scale.set(0.137, height * 1.005, 0.137);
          dummy.updateMatrix();
          outlines.setMatrixAt(index, dummy.matrix);

          const isLightPole = /light|lighting|fixture|lamp|luminaire/i.test(
            `${record.type} ${remarks}`,
          );

          if (isLightPole) {
            lightPoleTransforms.push({
              x: record.local.x,
              y: baseY + height,
              z: record.local.z,
              baseY,
              height,
            });
          }
        });

        poles.instanceMatrix.needsUpdate = true;
        outlines.instanceMatrix.needsUpdate = true;
        if (["structures", "tse"].includes(layerKey)) {
          poles.renderOrder = 30;
          outlines.renderOrder = 31;
        }
        group.add(poles, outlines);

        if (lightPoleTransforms.length) {
          const bulbGeometry = new THREE.SphereGeometry(1, 10, 8);
          const bulbMaterial = new THREE.MeshBasicMaterial({
            color: 0xfff7d6,
            transparent: true,
            opacity: 1,
            depthWrite: false,
            blending: THREE.AdditiveBlending,
          });
          const bulbs = new THREE.InstancedMesh(
            bulbGeometry,
            bulbMaterial,
            lightPoleTransforms.length,
          );
          bulbs.name = "structure-light-bulbs";
          bulbs.renderOrder = 18;

          const innerHaloMaterial = new THREE.MeshBasicMaterial({
            color: 0xffe7a0,
            transparent: true,
            opacity: 0.2,
            depthWrite: false,
            depthTest: true,
            blending: THREE.AdditiveBlending,
          });
          const innerHalos = new THREE.InstancedMesh(
            bulbGeometry,
            innerHaloMaterial,
            lightPoleTransforms.length,
          );
          innerHalos.name = "structure-light-inner-halos";
          innerHalos.renderOrder = 17;

          const outerHaloMaterial = new THREE.MeshBasicMaterial({
            color: 0xffc95f,
            transparent: true,
            opacity: 0.055,
            depthWrite: false,
            depthTest: true,
            blending: THREE.AdditiveBlending,
          });
          const outerHalos = new THREE.InstancedMesh(
            bulbGeometry,
            outerHaloMaterial,
            lightPoleTransforms.length,
          );
          outerHalos.name = "structure-light-outer-halos";
          outerHalos.renderOrder = 16;

          const glowCanvas = document.createElement("canvas");
          glowCanvas.width = 128;
          glowCanvas.height = 128;
          const glowContext = glowCanvas.getContext("2d");
          const gradient = glowContext.createRadialGradient(
            64,
            64,
            0,
            64,
            64,
            64,
          );
          gradient.addColorStop(0, "rgba(255,250,220,0.78)");
          gradient.addColorStop(0.16, "rgba(255,222,132,0.38)");
          gradient.addColorStop(0.5, "rgba(255,183,62,0.09)");
          gradient.addColorStop(1, "rgba(255,140,0,0)");
          glowContext.fillStyle = gradient;
          glowContext.fillRect(0, 0, 128, 128);

          const glowTexture = new THREE.CanvasTexture(glowCanvas);
          glowTexture.needsUpdate = true;
          const groundGlowMaterial = new THREE.MeshBasicMaterial({
            map: glowTexture,
            color: 0xffd67a,
            transparent: true,
            opacity: 0.14,
            depthWrite: false,
            depthTest: true,
            blending: THREE.AdditiveBlending,
            side: THREE.DoubleSide,
          });
          const groundGlowGeometry = new THREE.PlaneGeometry(1, 1);
          const groundGlows = new THREE.InstancedMesh(
            groundGlowGeometry,
            groundGlowMaterial,
            lightPoleTransforms.length,
          );
          groundGlows.name = "structure-light-ground-pools";
          groundGlows.renderOrder = 15;

          lightPoleTransforms.forEach((light, index) => {
            dummy.position.set(light.x, light.y, light.z);
            dummy.rotation.set(0, 0, 0);
            dummy.scale.setScalar(0.22);
            dummy.updateMatrix();
            bulbs.setMatrixAt(index, dummy.matrix);

            dummy.scale.setScalar(0.58);
            dummy.updateMatrix();
            innerHalos.setMatrixAt(index, dummy.matrix);

            dummy.scale.setScalar(1.08);
            dummy.updateMatrix();
            outerHalos.setMatrixAt(index, dummy.matrix);

            const poolDiameter = Math.max(
              3.5,
              Math.min(8, light.height * 0.82),
            );
            dummy.position.set(light.x, light.baseY + 0.035, light.z);
            dummy.rotation.set(-Math.PI / 2, 0, 0);
            dummy.scale.set(poolDiameter, poolDiameter, 1);
            dummy.updateMatrix();
            groundGlows.setMatrixAt(index, dummy.matrix);
          });

          bulbs.instanceMatrix.needsUpdate = true;
          innerHalos.instanceMatrix.needsUpdate = true;
          outerHalos.instanceMatrix.needsUpdate = true;
          groundGlows.instanceMatrix.needsUpdate = true;
          group.add(bulbs, outerHalos, innerHalos, groundGlows);

          // Do not add real PointLight objects here. With hundreds of pale
          // holographic assets, overlapping physical lights wash the whole park
          // toward white. The emissive bulb, two compact halos, and localized
          // ground pool still identify each lighting pole without recolouring
          // vegetation, paths, structures, or terrain around it.
        }
      }

      if (manholeRecords.length) {
        const geometry = new THREE.CylinderGeometry(1, 1, 0.12, 14, 1);
        const manholes = new THREE.InstancedMesh(
          geometry,
          materials.solid,
          manholeRecords.length,
        );
        const outlines = new THREE.InstancedMesh(
          geometry,
          materials.wire,
          manholeRecords.length,
        );

        manholeRecords.forEach((record, index) => {
          const baseY =
            this.getStandaloneTerrainHeightAtPoint(record.local, modelData) +
            0.07;
          dummy.position.set(record.local.x, baseY, record.local.z);
          dummy.quaternion.identity();
          dummy.scale.set(0.52, 1, 0.52);
          dummy.updateMatrix();
          manholes.setMatrixAt(index, dummy.matrix);
          dummy.scale.set(0.55, 1.03, 0.55);
          dummy.updateMatrix();
          outlines.setMatrixAt(index, dummy.matrix);
        });

        manholes.instanceMatrix.needsUpdate = true;
        outlines.instanceMatrix.needsUpdate = true;
        if (["structures", "tse"].includes(layerKey)) {
          manholes.renderOrder = 30;
          outlines.renderOrder = 31;
        }
        group.add(manholes, outlines);
      }

      if (markerRecords.length && layerKey === "structures") {
        const remainingMarkers = [];
        markerRecords.forEach((record) => {
          const style = this.getStandaloneDetailedEquipmentStyle(
            record.featureType || record.type,
            record.properties,
          );
          if (!style) {
            remainingMarkers.push(record);
            return;
          }
          const detailGroup = this.createStandaloneDetailedEquipmentPointGroup(
            record.local,
            modelData,
            style,
          );
          if (detailGroup) group.add(detailGroup);
          else remainingMarkers.push(record);
        });
        markerRecords.length = 0;
        markerRecords.push(...remainingMarkers);
      }

      if (markerRecords.length) {
        const geometry = new THREE.BoxGeometry(0.7, 0.7, 0.7);
        const markers = new THREE.InstancedMesh(
          geometry,
          materials.solid,
          markerRecords.length,
        );
        const outlines = new THREE.InstancedMesh(
          geometry,
          materials.wire,
          markerRecords.length,
        );

        markerRecords.forEach((record, index) => {
          const baseY =
            this.getStandaloneTerrainHeightAtPoint(record.local, modelData) +
            (layerKey === "water" ? 0.92 : 0.4);
          dummy.position.set(record.local.x, baseY, record.local.z);
          dummy.quaternion.identity();
          dummy.scale.setScalar(1);
          dummy.updateMatrix();
          markers.setMatrixAt(index, dummy.matrix);
          dummy.scale.setScalar(1.05);
          dummy.updateMatrix();
          outlines.setMatrixAt(index, dummy.matrix);
        });

        markers.instanceMatrix.needsUpdate = true;
        outlines.instanceMatrix.needsUpdate = true;
        if (layerKey === "water") {
          markers.renderOrder = 62;
          outlines.renderOrder = 63;
        } else if (layerKey === "structures") {
          markers.renderOrder = 30;
          outlines.renderOrder = 31;
        }
        group.add(markers, outlines);
      }

      group.userData.pointCount =
        poleRecords.length + manholeRecords.length + markerRecords.length;
      return group;
    },

    createStandaloneConceptPathMaterial(paletteOverride = null) {
      const palette =
        paletteOverride ||
        this.getStandaloneFeaturePalette("paths", "pedestrian", {
          CL_LAND_COVER_DETAILED: "Pedestrian Space",
        });
      return new THREE.ShaderMaterial({
        uniforms: {
          uBaseColor: { value: new THREE.Color(palette.solid) },
          uLineColor: { value: new THREE.Color(palette.line) },
          uCellX: { value: THREE_PATH_GRID_CELL_METERS },
          uCellZ: { value: THREE_PATH_GRID_CELL_METERS },
        },
        vertexShader: `
          varying vec3 vWorldPosition;
          void main() {
            vec4 worldPosition = modelMatrix * vec4(position, 1.0);
            vWorldPosition = worldPosition.xyz;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
          }
        `,
        fragmentShader: `
          uniform vec3 uBaseColor;
          uniform vec3 uLineColor;
          uniform float uCellX;
          uniform float uCellZ;
          varying vec3 vWorldPosition;
          void main() {
            vec2 scaled = vec2(vWorldPosition.x / uCellX, vWorldPosition.z / uCellZ);
            vec2 width = max(fwidth(scaled), vec2(0.0001));
            vec2 distanceToLine =
              abs(fract(scaled - 0.5) - 0.5) /
              (width * 0.48);
            float line =
              1.0 - clamp(min(distanceToLine.x, distanceToLine.y), 0.0, 1.0);
            line = smoothstep(0.08, 0.98, line);
            vec3 color = mix(
              uBaseColor,
              uLineColor,
              line * ${THREE_PATH_GRID_OPACITY}
            );
            gl_FragColor = vec4(color, 1.0);
          }
        `,
        transparent: false,
        depthTest: true,
        depthWrite: true,
        side: THREE.DoubleSide,
        polygonOffset: true,
        polygonOffsetFactor: -1,
        polygonOffsetUnits: -1,
        extensions: { derivatives: true },
      });
    },

    appendStandaloneConceptPathHandrails(
      feature,
      featureType,
      modelData,
      positions,
    ) {
      const properties = (feature && feature.properties) || {};
      const description = `${featureType || ""} ${
        properties.QC_REMARKS || ""
      } ${properties.Remarks || ""} ${properties.Type || ""}`.toLowerCase();
      if (!/stair|stairs|staircase|bridge|footbridge|ramp/.test(description)) {
        return;
      }

      this.getStandalonePolygonParts(feature.geometry).forEach((rings) => {
        const localRings = this.getStandaloneLocalPolygonRings(
          rings,
          modelData.origin,
        );
        const outer = localRings[0] || [];
        if (outer.length < 2) return;

        outer.forEach((point, index) => {
          const next = outer[(index + 1) % outer.length];
          const dx = next.x - point.x;
          const dz = next.y - point.y;
          const length = Math.hypot(dx, dz);
          if (length < 0.25) return;

          const divisions = Math.max(1, Math.ceil(length / 3));
          for (let step = 0; step <= divisions; step += 1) {
            const ratio = step / divisions;
            const x = point.x + dx * ratio;
            const z = point.y + dz * ratio;
            const ground =
              this.getStandaloneTerrainHeightAtPoint({ x, z }, modelData) +
              0.34;
            positions.push(x, ground, z, x, ground + 1.15, z);
          }

          const y1 =
            this.getStandaloneTerrainHeightAtPoint(
              { x: point.x, z: point.y },
              modelData,
            ) + 1.49;
          const y2 =
            this.getStandaloneTerrainHeightAtPoint(
              { x: next.x, z: next.y },
              modelData,
            ) + 1.49;
          positions.push(point.x, y1, point.y, next.x, y2, next.y);
        });
      });
    },

    createStandaloneConceptPathOverlay(geojson, modelData) {
      const group = new THREE.Group();
      group.name = "concept-path-overlay";
      group.userData.isConceptPathOverlay = true;
      const features =
        geojson && Array.isArray(geojson.features) ? geojson.features : [];
      const surfacePositions = [];
      const outlinePositions = [];
      const sidewalkSurfacePositions = [];
      const sidewalkOutlinePositions = [];
      const roadParkingSurfacePositions = [];
      const roadParkingOutlinePositions = [];
      const handrailPositions = [];

      features.forEach((feature) => {
        const featureType = this.getStandaloneFeatureType(feature);
        const properties = (feature && feature.properties) || {};
        const palette = this.getStandaloneFeaturePalette(
          "paths",
          featureType,
          properties,
        );
        const isRoadOrParking = palette.key === "path-road-parking";
        const isSidewalk = palette.key === "path-sidewalk";
        const targetSurfacePositions = isRoadOrParking
          ? roadParkingSurfacePositions
          : isSidewalk
            ? sidewalkSurfacePositions
            : surfacePositions;
        const targetOutlinePositions = isRoadOrParking
          ? roadParkingOutlinePositions
          : isSidewalk
            ? sidewalkOutlinePositions
            : outlinePositions;

        this.getStandalonePolygonParts(feature && feature.geometry).forEach(
          (rings) => {
            const localRings = this.getStandaloneLocalPolygonRings(
              rings,
              modelData.origin,
            );
            if (!localRings.length) return;
            this.appendStandaloneFlatPolygonGeometry(
              localRings,
              modelData,
              THREE_PATH_TERRAIN_CLEARANCE_METERS,
              targetSurfacePositions,
              targetOutlinePositions,
              {
                preventTerrainOcclusion: false,
                drapeToTerrain: true,
                drapeSegmentLength: THREE_PATH_DRAPE_SEGMENT_METERS,
                maxDrapeSubdivisions: THREE_PATH_DRAPE_MAX_SUBDIVISIONS,
                terrainClearance: 0.02,
              },
            );
          },
        );

        if (!isRoadOrParking) {
          this.appendStandaloneConceptPathHandrails(
            feature,
            featureType,
            modelData,
            handrailPositions,
          );
        }
      });

      if (roadParkingSurfacePositions.length) {
        const palette = this.getStandaloneFeaturePalette("paths", "road", {
          CL_ROAD_TYPE: "Carriageway",
        });
        const roadMaterials = this.createStandaloneLayerMaterials(
          "paths",
          palette,
        );
        const geometry = new THREE.BufferGeometry();
        geometry.setAttribute(
          "position",
          new THREE.Float32BufferAttribute(roadParkingSurfacePositions, 3),
        );
        geometry.computeVertexNormals();
        const mesh = new THREE.Mesh(geometry, roadMaterials.solid);
        mesh.name = "concept-road-parking-surfaces";
        mesh.renderOrder = 6;
        mesh.frustumCulled = false;
        group.add(mesh);

        if (roadParkingOutlinePositions.length) {
          const outlineGeometry = new THREE.BufferGeometry();
          outlineGeometry.setAttribute(
            "position",
            new THREE.Float32BufferAttribute(roadParkingOutlinePositions, 3),
          );
          const outlines = new THREE.LineSegments(
            outlineGeometry,
            roadMaterials.lines,
          );
          outlines.name = "concept-road-parking-outlines";
          outlines.renderOrder = 7;
          group.add(outlines);
        }
      }

      if (sidewalkSurfacePositions.length) {
        const palette = this.getStandaloneFeaturePalette("paths", "sidewalk", {
          CL_ROAD_TYPE: "Sidewalk",
        });
        const sidewalkMaterials = this.createStandaloneLayerMaterials(
          "paths",
          palette,
        );
        const geometry = new THREE.BufferGeometry();
        geometry.setAttribute(
          "position",
          new THREE.Float32BufferAttribute(sidewalkSurfacePositions, 3),
        );
        geometry.computeVertexNormals();
        const mesh = new THREE.Mesh(geometry, sidewalkMaterials.solid);
        mesh.name = "concept-sidewalk-surfaces";
        mesh.renderOrder = 5;
        mesh.frustumCulled = false;
        group.add(mesh);

        if (sidewalkOutlinePositions.length) {
          const outlineGeometry = new THREE.BufferGeometry();
          outlineGeometry.setAttribute(
            "position",
            new THREE.Float32BufferAttribute(sidewalkOutlinePositions, 3),
          );
          const outlines = new THREE.LineSegments(
            outlineGeometry,
            sidewalkMaterials.lines,
          );
          outlines.name = "concept-sidewalk-outlines";
          outlines.renderOrder = 6;
          group.add(outlines);
        }
      }

      if (surfacePositions.length) {
        const pedestrianPalette = this.getStandaloneFeaturePalette(
          "paths",
          "pedestrian",
          { CL_LAND_COVER_DETAILED: "Pedestrian Space" },
        );
        const geometry = new THREE.BufferGeometry();
        geometry.setAttribute(
          "position",
          new THREE.Float32BufferAttribute(surfacePositions, 3),
        );
        geometry.computeVertexNormals();
        const mesh = new THREE.Mesh(
          geometry,
          this.createStandaloneConceptPathMaterial(pedestrianPalette),
        );
        mesh.name = "concept-path-grid-surfaces";
        mesh.renderOrder = 6;
        mesh.frustumCulled = false;
        group.add(mesh);
      }

      const lineMaterial = new THREE.LineBasicMaterial({
        color: THREE_PATH_PEDESTRIAN_LINE_COLOR,
        transparent: true,
        opacity: 0.34,
        depthWrite: false,
      });
      if (outlinePositions.length) {
        const geometry = new THREE.BufferGeometry();
        geometry.setAttribute(
          "position",
          new THREE.Float32BufferAttribute(outlinePositions, 3),
        );
        const outlines = new THREE.LineSegments(geometry, lineMaterial);
        outlines.name = "concept-path-outlines";
        outlines.renderOrder = 7;
        group.add(outlines);
      }
      if (handrailPositions.length) {
        const geometry = new THREE.BufferGeometry();
        geometry.setAttribute(
          "position",
          new THREE.Float32BufferAttribute(handrailPositions, 3),
        );
        const handrails = new THREE.LineSegments(
          geometry,
          lineMaterial.clone(),
        );
        handrails.material.opacity = 1;
        handrails.name = "concept-path-handrails";
        handrails.renderOrder = 16;
        group.add(handrails);
      }

      group.visible = false;
      return group;
    },

    applyStandaloneConceptPathMode(enabled = this.threeToolState.conceptPaths) {
      const pathsGroup = this.threeLayerGroups && this.threeLayerGroups.paths;
      if (!pathsGroup) return;
      const layerVisible = Boolean(this.threeLayerVisibility.paths);
      const subLayerVisibility = this.threeSubLayerVisibility.paths || {};
      const renderGroups = pathsGroup.children.some(
        (child) => child && child.userData && child.userData.subLayerKey,
      )
        ? pathsGroup.children
        : [pathsGroup];

      renderGroups.forEach((renderGroup) => {
        const subLayerKey =
          renderGroup &&
          renderGroup.userData &&
          renderGroup.userData.subLayerKey;
        const subLayerVisible =
          !subLayerKey || subLayerVisibility[subLayerKey] !== false;
        renderGroup.visible = layerVisible && subLayerVisible;

        const isPedestrianSpace =
          String(subLayerKey || "").toLowerCase() === "pedestrian-space";

        renderGroup.children.forEach((child) => {
          const isConcept = Boolean(
            child && child.userData && child.userData.isConceptPathOverlay,
          );
          const keepInConceptMode = Boolean(
            child && child.userData && child.userData.keepInConceptPathMode,
          );
          const childName = String((child && child.name) || "").toLowerCase();
          const isNormalPolygonContainer =
            childName === "paths-polygon-container";

          if (isConcept) {
            // Pedestrian Space always uses this draped grid surface. For other
            // PATHS sub-layers it remains tied to the optional concept tool.
            child.visible =
              (Boolean(enabled) || isPedestrianSpace) &&
              layerVisible &&
              subLayerVisible;
          } else if (isPedestrianSpace && isNormalPolygonContainer) {
            // Avoid z-fighting: the always-on Pedestrian Space grid surface
            // replaces the plain polygon fill while preserving point/line
            // objects belonging to this same sub-layer.
            child.visible = false;
          } else {
            child.visible = keepInConceptMode
              ? layerVisible && subLayerVisible
              : !enabled && layerVisible && subLayerVisible;
          }
        });
      });
    },

    createStandaloneLineInstanceObjects(
      layerKey,
      records,
      modelData,
      materials,
    ) {
      const group = new THREE.Group();
      const segments = [];

      records.forEach(({ feature, featureType }) => {
        const geometry = feature && feature.geometry;
        const properties = (feature && feature.properties) || {};
        this.getStandaloneLineStrings(geometry).forEach((line) => {
          for (let index = 0; index < line.length - 1; index += 1) {
            const startCoordinate = line[index];
            const endCoordinate = line[index + 1];
            if (
              !Array.isArray(startCoordinate) ||
              !Array.isArray(endCoordinate)
            ) {
              continue;
            }
            const startLocal = this.lngLatToTerrainMeters(
              startCoordinate,
              modelData.origin,
            );
            const endLocal = this.lngLatToTerrainMeters(
              endCoordinate,
              modelData.origin,
            );
            const midpoint = {
              x: (startLocal.x + endLocal.x) / 2,
              z: (startLocal.z + endLocal.z) / 2,
            };
            const diameterMillimetres = this.getStandaloneNumericProperty(
              properties,
              ["Diameter", "diameter"],
              0,
            );
            const utilityLabel =
              layerKey === "tse"
                ? this.getStandaloneUtilitySubLayerLabel(feature)
                : "";
            const radius =
              layerKey === "tse" && utilityLabel !== "Powerline"
                ? Math.max(
                    0.18,
                    Math.min(0.48, diameterMillimetres / 2000 || 0.24),
                  )
                : /power/.test(featureType) || utilityLabel === "Powerline"
                  ? 0.1
                  : 0.12;
            const aboveGround =
              String(properties.is_aboveground || "").toUpperCase() === "Y";
            segments.push({
              startLocal,
              endLocal,
              radius,
              heightOffset: aboveGround
                ? 2.5
                : layerKey === "water"
                  ? 0.9
                  : layerKey === "tse" && utilityLabel !== "Powerline"
                    ? 0.32
                    : 0.2,
            });
          }
        });
      });

      if (!segments.length) return group;

      const geometry = new THREE.CylinderGeometry(1, 1, 1, 7, 1, false);
      const mesh = new THREE.InstancedMesh(
        geometry,
        materials.solid,
        segments.length,
      );
      const outlines = new THREE.InstancedMesh(
        geometry,
        materials.wire,
        segments.length,
      );
      const dummy = new THREE.Object3D();
      const up = new THREE.Vector3(0, 1, 0);
      const start = new THREE.Vector3();
      const end = new THREE.Vector3();
      const direction = new THREE.Vector3();

      segments.forEach((segment, index) => {
        start.set(
          segment.startLocal.x,
          this.getStandaloneTerrainHeightAtPoint(
            segment.startLocal,
            modelData,
          ) + segment.heightOffset,
          segment.startLocal.z,
        );
        end.set(
          segment.endLocal.x,
          this.getStandaloneTerrainHeightAtPoint(segment.endLocal, modelData) +
            segment.heightOffset,
          segment.endLocal.z,
        );
        direction.copy(end).sub(start);
        const length = Math.max(0.01, direction.length());

        dummy.position.copy(start).add(end).multiplyScalar(0.5);
        dummy.quaternion.setFromUnitVectors(up, direction.normalize());
        dummy.scale.set(segment.radius, length, segment.radius);
        dummy.updateMatrix();
        mesh.setMatrixAt(index, dummy.matrix);

        dummy.scale.set(
          segment.radius * 1.08,
          length * 1.005,
          segment.radius * 1.08,
        );
        dummy.updateMatrix();
        outlines.setMatrixAt(index, dummy.matrix);
      });

      mesh.instanceMatrix.needsUpdate = true;
      outlines.instanceMatrix.needsUpdate = true;
      if (layerKey === "water") {
        mesh.renderOrder = 62;
        outlines.renderOrder = 63;
      }
      group.add(mesh, outlines);
      group.userData.segmentCount = segments.length;
      return group;
    },

    async createStandaloneGenericFeatureGroup(
      layerKey,
      geojson,
      modelData,
      contextGeoJson = geojson,
    ) {
      const group = new THREE.Group();
      group.name = `standalone-${layerKey}`;
      const materials = this.createStandaloneLayerMaterials(layerKey);
      const features =
        geojson && Array.isArray(geojson.features) ? geojson.features : [];
      const pointRecords = [];
      const lineRecords = [];
      const polygonRecords = [];
      const contextFeatures =
        contextGeoJson && Array.isArray(contextGeoJson.features)
          ? contextGeoJson.features
          : features;
      const contextPointRecords = [];
      const contextPolygonRecords = [];

      contextFeatures.forEach((feature) => {
        const geometry = feature && feature.geometry;
        if (!geometry) return;
        const record = {
          feature,
          featureType: this.getStandaloneFeatureType(feature),
        };

        if (layerKey === "structures") {
          const explicitLayer = this.getStandaloneFeatureLayer(feature);
          const fallbackLayer = this.normalizeStandaloneLayerFilter(
            this.threeProjectGisFallbackLayer,
          );
          const effectiveLayer = explicitLayer || fallbackLayer;
          const utilityLabel = this.getStandaloneUtilitySubLayerLabel(feature);

          if (geometry.type === "Point" || geometry.type === "MultiPoint") {
            // Keep utility poles in the structure context so covered pergolas
            // still raise above them, even though those poles are controlled
            // from the UTILITY filter instead of STRUCTURES.
            if (
              effectiveLayer === "structures" ||
              utilityLabel === "Lighting" ||
              utilityLabel === "Electric Pole"
            ) {
              contextPointRecords.push(record);
            }
            return;
          }

          if (
            (geometry.type === "Polygon" || geometry.type === "MultiPolygon") &&
            effectiveLayer === "structures"
          ) {
            contextPolygonRecords.push(record);
          }
          return;
        }

        if (geometry.type === "Point" || geometry.type === "MultiPoint") {
          contextPointRecords.push(record);
        } else if (
          geometry.type === "Polygon" ||
          geometry.type === "MultiPolygon"
        ) {
          contextPolygonRecords.push(record);
        }
      });

      features.forEach((feature) => {
        const geometry = feature && feature.geometry;
        if (!geometry) return;
        const record = {
          feature,
          featureType: this.getStandaloneFeatureType(feature),
        };
        if (geometry.type === "Point" || geometry.type === "MultiPoint") {
          pointRecords.push(record);
        } else if (
          geometry.type === "LineString" ||
          geometry.type === "MultiLineString"
        ) {
          lineRecords.push(record);
        } else if (
          geometry.type === "Polygon" ||
          geometry.type === "MultiPolygon"
        ) {
          polygonRecords.push(record);
        }
      });

      const polygonContainer = new THREE.Group();
      polygonContainer.name = `${layerKey}-polygon-container`;
      let polygonCount = 0;
      let halfBasketballCourtCount = 0;
      let fullBasketballCourtCount = 0;
      let mugaCourtCount = 0;
      let skateparkCount = 0;

      if (["vegetation", "paths"].includes(layerKey) && polygonRecords.length) {
        const polygonBuckets = new Map();

        polygonRecords.forEach((record) => {
          const properties =
            (record.feature && record.feature.properties) || {};
          const palette = this.getStandaloneFeaturePalette(
            layerKey,
            record.featureType,
            properties,
          );
          const key =
            palette.key ||
            `${layerKey === "vegetation" ? "vegetation" : "path"}-default`;
          if (!polygonBuckets.has(key)) {
            polygonBuckets.set(key, { palette, records: [] });
          }
          polygonBuckets.get(key).records.push(record);
        });

        polygonBuckets.forEach(({ palette, records }) => {
          const bucketMaterials = this.createStandaloneLayerMaterials(
            layerKey,
            palette,
          );
          const bucketGroup = this.createStandaloneMergedPolygonObjects(
            layerKey,
            records,
            modelData,
            bucketMaterials,
            pointRecords,
            contextPolygonRecords,
            contextPointRecords,
          );
          bucketGroup.name = `${layerKey}-${palette.key || "default"}`;
          bucketGroup.userData.pathPaletteKey =
            layerKey === "paths" ? palette.key || "" : "";
          polygonCount += bucketGroup.userData.polygonCount || 0;
          halfBasketballCourtCount +=
            bucketGroup.userData.halfBasketballCourtCount || 0;
          fullBasketballCourtCount +=
            bucketGroup.userData.fullBasketballCourtCount || 0;
          mugaCourtCount += bucketGroup.userData.mugaCourtCount || 0;
          skateparkCount += bucketGroup.userData.skateparkCount || 0;
          polygonContainer.add(bucketGroup);
        });
      } else {
        const polygonGroup = this.createStandaloneMergedPolygonObjects(
          layerKey,
          polygonRecords,
          modelData,
          materials,
          pointRecords,
          contextPolygonRecords,
          contextPointRecords,
        );
        polygonCount = polygonGroup.userData.polygonCount || 0;
        halfBasketballCourtCount =
          polygonGroup.userData.halfBasketballCourtCount || 0;
        fullBasketballCourtCount =
          polygonGroup.userData.fullBasketballCourtCount || 0;
        mugaCourtCount = polygonGroup.userData.mugaCourtCount || 0;
        skateparkCount = polygonGroup.userData.skateparkCount || 0;
        polygonContainer.add(polygonGroup);
      }

      const pointGroup = this.createStandalonePointInstanceObjects(
        pointRecords,
        modelData,
        materials,
        layerKey,
      );

      let lineGroup;
      if (layerKey === "paths" && lineRecords.length) {
        lineGroup = new THREE.Group();
        lineGroup.name = "paths-line-container";
        lineGroup.userData.keepInConceptPathMode = true;
        let segmentCount = 0;
        const lineBuckets = new Map();

        lineRecords.forEach((record) => {
          const properties =
            (record.feature && record.feature.properties) || {};
          const palette = this.getStandaloneFeaturePalette(
            layerKey,
            record.featureType,
            properties,
          );
          const key = palette.key || "path-pedestrian";
          if (!lineBuckets.has(key)) {
            lineBuckets.set(key, { palette, records: [] });
          }
          lineBuckets.get(key).records.push(record);
        });

        lineBuckets.forEach(({ palette, records }) => {
          const bucketMaterials = this.createStandaloneLayerMaterials(
            layerKey,
            palette,
          );
          const bucketGroup = this.createStandaloneLineInstanceObjects(
            layerKey,
            records,
            modelData,
            bucketMaterials,
          );
          bucketGroup.name = `paths-lines-${palette.key || "default"}`;
          bucketGroup.userData.pathPaletteKey = palette.key || "";
          segmentCount += bucketGroup.userData.segmentCount || 0;
          lineGroup.add(bucketGroup);
        });

        lineGroup.userData.segmentCount = segmentCount;
      } else {
        lineGroup = this.createStandaloneLineInstanceObjects(
          layerKey,
          lineRecords,
          modelData,
          materials,
        );
      }

      group.add(polygonContainer, pointGroup, lineGroup);

      if (layerKey === "paths") {
        const conceptOverlay = this.createStandaloneConceptPathOverlay(
          geojson,
          modelData,
        );
        group.add(conceptOverlay);
      }

      group.userData.featureCount = features.length;
      group.userData.polygonCount = polygonCount;
      group.userData.halfBasketballCourtCount = halfBasketballCourtCount;
      group.userData.fullBasketballCourtCount = fullBasketballCourtCount;
      group.userData.mugaCourtCount = mugaCourtCount;
      group.userData.skateparkCount = skateparkCount;
      group.userData.detailedModularBuildingCount =
        polygonContainer.children.reduce(
          (count, child) =>
            count + Number(child?.userData?.detailedModularBuildingCount || 0),
          0,
        );
      group.userData.pointCount = pointGroup.userData.pointCount || 0;
      group.userData.segmentCount = lineGroup.userData.segmentCount || 0;
      return group;
    },

    async ensureStandaloneGenericFeatureLayer(layerKey, url) {
      if (!this.threeScene || !this.threeParkModelData) {
        this.$set(this.threeLayerVisibility, layerKey, false);
        return;
      }

      const existing = this.threeLayerGroups && this.threeLayerGroups[layerKey];
      if (existing) {
        this.makeStandaloneLayerGroupOpaque(existing);
        existing.visible = true;
        return;
      }

      this.threeLayerLoadingKey = layerKey;
      this.threeLayerError = "";

      try {
        const sourceGeoJson =
          this.threeProjectGisData &&
          this.threeProjectGisUrl === String(url || "")
            ? this.threeProjectGisData
            : await this.loadStandaloneGeoJson(url);
        const geojson = this.filterStandaloneGeoJsonByLayer(
          sourceGeoJson,
          layerKey,
        );
        const group = await this.createStandaloneSubLayeredGenericFeatureGroup(
          layerKey,
          geojson,
          this.threeParkModelData,
          layerKey === "structures" ? sourceGeoJson : geojson,
        );

        if (!this.isStandaloneThreeMode) {
          this.disposeStandaloneThreeGroup(group);
          return;
        }

        this.makeStandaloneLayerGroupOpaque(group);
        group.visible = Boolean(this.threeLayerVisibility[layerKey]);
        this.threeScene.add(group);
        this.threeLayerGroups[layerKey] = group;
        this.applyStandaloneSubLayerVisibility(layerKey);

        if (layerKey === "paths") {
          this.applyStandaloneConceptPathMode(this.threeToolState.conceptPaths);
        }

        if (layerKey === "structures") {
          // The ArcGIS-exported structure polygons contain the department's
          // actual facility classifications. Do not mix in generic Mapbox
          // "Building" labels because they can occupy the same screen cells
          // and suppress Sports Courts, Restaurant, Bathrooms, Prayer Room, etc.
          this.threeStructureAnnotations =
            this.buildStandaloneStructureAnnotations(
              geojson,
              this.threeParkModelData,
            );
          this.updateStandaloneStructureLabelOverlay(true);
        }

        console.info(`3D ${layerKey} diagnostics:`, {
          source: url,
          featureCount: group.userData.featureCount,
          polygonCount: group.userData.polygonCount,
          halfBasketballCourtCount:
            group.userData.halfBasketballCourtCount || 0,
          fullBasketballCourtCount:
            group.userData.fullBasketballCourtCount || 0,
          mugaCourtCount: group.userData.mugaCourtCount || 0,
          skateparkCount: group.userData.skateparkCount || 0,
          detailedModularBuildingCount:
            group.userData.detailedModularBuildingCount || 0,
          pointCount: group.userData.pointCount,
          segmentCount: group.userData.segmentCount,
        });
      } catch (error) {
        console.error(`The 3D ${layerKey} layer could not be created:`, error);
        this.$set(this.threeLayerVisibility, layerKey, false);
        this.threeLayerError = this.isArabicUI()
          ? "تعذر تحميل طبقة الخريطة ثلاثية الأبعاد."
          : `The 3D ${layerKey} layer could not be loaded.`;
      } finally {
        this.threeLayerLoadingKey = "";
      }
    },

    disposeStandaloneThreeGroup(group) {
      if (!group || typeof group.traverse !== "function") return;

      group.traverse((object) => {
        if (object.geometry && object.geometry.dispose) {
          object.geometry.dispose();
        }

        const materials = Array.isArray(object.material)
          ? object.material
          : object.material
            ? [object.material]
            : [];
        materials.forEach((material) => {
          if (material && material.dispose) material.dispose();
        });
      });

      if (group.parent) group.parent.remove(group);
    },

    async ensureStandaloneVegetationLayer(url) {
      if (!this.threeScene || !this.threeParkModelData) {
        this.$set(this.threeLayerVisibility, "vegetation", false);
        return;
      }

      const existing =
        this.threeLayerGroups && this.threeLayerGroups.vegetation;
      if (existing) {
        this.makeStandaloneLayerGroupOpaque(existing);
        existing.visible = true;
        this.applyStandaloneSubLayerVisibility("vegetation");
        return;
      }

      const loadToken = this.threeVegetationLoadToken + 1;
      this.threeVegetationLoadToken = loadToken;
      this.threeLayerLoadingKey = "vegetation";
      this.threeLayerError = "";

      try {
        const sourceGeoJson =
          this.threeProjectGisData &&
          this.threeProjectGisUrl === String(url || "")
            ? this.threeProjectGisData
            : await this.loadStandaloneGeoJson(url);
        const geojson = this.filterStandaloneGeoJsonByLayer(
          sourceGeoJson,
          "vegetation",
        );
        if (
          loadToken !== this.threeVegetationLoadToken ||
          !this.isStandaloneThreeMode
        ) {
          return;
        }

        await new Promise((resolve) => window.requestAnimationFrame(resolve));
        const vegetationGroup =
          await this.createStandaloneSubLayeredVegetationGroup(
            geojson,
            this.threeParkModelData,
            loadToken,
          );

        if (
          loadToken !== this.threeVegetationLoadToken ||
          !this.isStandaloneThreeMode
        ) {
          this.disposeStandaloneThreeGroup(vegetationGroup);
          return;
        }

        this.makeStandaloneLayerGroupOpaque(vegetationGroup);
        vegetationGroup.visible = Boolean(this.threeLayerVisibility.vegetation);
        this.threeScene.add(vegetationGroup);
        this.threeLayerGroups.vegetation = vegetationGroup;
        this.applyStandaloneSubLayerVisibility("vegetation");

        console.info("3D vegetation diagnostics:", {
          source: url,
          treeCount: vegetationGroup.userData.treeCount,
          speciesCount: vegetationGroup.userData.speciesCount,
          instancedBuckets: vegetationGroup.userData.bucketCount,
        });
      } catch (error) {
        if (loadToken !== this.threeVegetationLoadToken) return;
        console.error("The 3D vegetation layer could not be created:", error);
        this.$set(this.threeLayerVisibility, "vegetation", false);
        this.threeLayerError = this.isArabicUI()
          ? "تعذر تحميل طبقة النباتات ثلاثية الأبعاد."
          : "The 3D vegetation layer could not be loaded.";
      } finally {
        if (loadToken === this.threeVegetationLoadToken) {
          this.threeLayerLoadingKey = "";
        }
      }
    },

    getQadisiyah3ProjectItem() {
      const rows = Array.isArray(this.$root?.data_src)
        ? this.$root.data_src
        : [];
      return rows.find((item) => this.isQadisiyah3ProjectItem(item)) || null;
    },

    isQadisiyah3ProjectActive() {
      if (this.activeViewMode !== "project" || !this.currentProjectId)
        return false;
      if (
        this.normalizeProjectId(this.currentProjectId) ===
        QADISIYAH_3_PROJECT_ID
      ) {
        return true;
      }

      const currentItem = this.getProjectItemById(this.currentProjectId);
      return this.isQadisiyah3ProjectItem(currentItem || {});
    },

    ensureConceptModeAllowed() {
      if (this.activeBaseMapMode === "3d" && !this.isProjectDetailsMap) {
        this.activeBaseMapMode = "street";
        this.exitStandaloneThreeMode();

        if (this.$map) {
          this.suspendMapStyleRestoration = false;
          this.mapboxRendererSuspended = false;
          this.projectLayersReady = false;
          this.unbindProjectLayerEvents();
          this.$map.setStyle(this.getMapboxStyleForMode("street"));
        }
      }

      if (this.isConceptMapMode() && !this.isQadisiyah3ProjectActive()) {
        this.activeBaseMapMode = "street";
        this.removeConceptMapLayers();
      }
    },

    getConceptImageList(item = null) {
      const projectItem = item || this.getQadisiyah3ProjectItem();
      return this.parseImagePathList(projectItem?.img_infodisplay);
    },

    getConceptLayerImagePath(layer, images = null) {
      if (!layer) return "";
      const imageList = Array.isArray(images)
        ? images
        : this.getConceptImageList();
      const imageFromProject = imageList[layer.imageIndex];
      return String(imageFromProject || layer.fallbackImage || "").trim();
    },

    getConceptLayerCrop(layer = null) {
      return (
        QADISIYAH_3_CONCEPT_SHARED_CROP ||
        layer?.crop ||
        QADISIYAH_3_CONCEPT_DEFAULT_CROP
      );
    },

    getConceptCropCacheKey(imageUrl, layer = null) {
      const crop = this.getConceptLayerCrop(layer);
      return [
        String(imageUrl || "").trim(),
        layer?.id || "default",
        crop.left,
        crop.top,
        crop.right,
        crop.bottom,
      ].join("|");
    },

    getClampedConceptCropValue(value) {
      const numberValue = Number(value);
      if (!Number.isFinite(numberValue)) return 0;
      return Math.min(0.45, Math.max(0, numberValue));
    },

    async getCroppedConceptImageUrl(imageUrl, layer = null) {
      const rawImageUrl = String(imageUrl || "").trim();
      if (
        !rawImageUrl ||
        typeof window === "undefined" ||
        typeof document === "undefined"
      ) {
        return rawImageUrl;
      }

      const crop = this.getConceptLayerCrop(layer);
      const cropValues = [crop.left, crop.top, crop.right, crop.bottom].map(
        this.getClampedConceptCropValue,
      );
      if (!cropValues.some((value) => value > 0)) return rawImageUrl;

      const cacheKey = this.getConceptCropCacheKey(rawImageUrl, layer);
      if (this.conceptImageUrlCache?.[cacheKey]) {
        return this.conceptImageUrlCache[cacheKey];
      }

      try {
        const croppedImageUrl = await this.cropConceptImageToDataUrl(
          rawImageUrl,
          crop,
        );
        if (croppedImageUrl) {
          this.conceptImageUrlCache[cacheKey] = croppedImageUrl;
          return croppedImageUrl;
        }
      } catch (error) {
        console.warn(
          "3D Concept image crop failed; using original sheet:",
          error,
        );
      }

      return rawImageUrl;
    },

    cropConceptImageToDataUrl(
      imageUrl,
      crop = QADISIYAH_3_CONCEPT_DEFAULT_CROP,
    ) {
      return new Promise((resolve, reject) => {
        const image = new Image();
        image.crossOrigin = "anonymous";
        image.onload = () => {
          try {
            const sourceWidth = image.naturalWidth || image.width;
            const sourceHeight = image.naturalHeight || image.height;
            if (!sourceWidth || !sourceHeight) {
              resolve(imageUrl);
              return;
            }

            const left = Math.round(
              sourceWidth * this.getClampedConceptCropValue(crop.left),
            );
            const top = Math.round(
              sourceHeight * this.getClampedConceptCropValue(crop.top),
            );
            const right = Math.round(
              sourceWidth * this.getClampedConceptCropValue(crop.right),
            );
            const bottom = Math.round(
              sourceHeight * this.getClampedConceptCropValue(crop.bottom),
            );
            const cropWidth = Math.max(1, sourceWidth - left - right);
            const cropHeight = Math.max(1, sourceHeight - top - bottom);

            const canvas = document.createElement("canvas");
            canvas.width = cropWidth;
            canvas.height = cropHeight;
            const context = canvas.getContext("2d");
            if (!context) {
              resolve(imageUrl);
              return;
            }

            const maskPoints = Array.isArray(
              QADISIYAH_3_CONCEPT_IMAGE_MASK_POINTS,
            )
              ? QADISIYAH_3_CONCEPT_IMAGE_MASK_POINTS
              : [];

            context.save();
            if (maskPoints.length >= 3) {
              context.beginPath();
              maskPoints.forEach((point, index) => {
                const normalizedX = Array.isArray(point) ? Number(point[0]) : 0;
                const normalizedY = Array.isArray(point) ? Number(point[1]) : 0;
                const x = cropWidth * Math.min(1, Math.max(0, normalizedX));
                const y = cropHeight * Math.min(1, Math.max(0, normalizedY));
                if (index === 0) context.moveTo(x, y);
                else context.lineTo(x, y);
              });
              context.closePath();
              context.clip();
            }

            context.drawImage(
              image,
              left,
              top,
              cropWidth,
              cropHeight,
              0,
              0,
              cropWidth,
              cropHeight,
            );
            context.restore();
            resolve(canvas.toDataURL("image/png"));
          } catch (error) {
            reject(error);
          }
        };
        image.onerror = reject;
        image.src = imageUrl;
      });
    },

    getSelectedConceptLayer() {
      const layers = this.visibleConceptMapLayers.length
        ? this.visibleConceptMapLayers
        : QADISIYAH_3_CONCEPT_LAYERS;
      return (
        layers.find((layer) => layer.id === this.selectedConceptLayerId) ||
        layers[0] ||
        QADISIYAH_3_CONCEPT_LAYERS[0]
      );
    },

    async setConceptMapLayer(layerId) {
      if (!QADISIYAH_3_CONCEPT_LAYERS.some((layer) => layer.id === layerId))
        return;
      this.selectedConceptLayerId = layerId;
      await this.syncConceptMapLayers();
    },

    getProjectFeaturesForId(projectId) {
      const id = this.normalizeProjectId(projectId);
      return this.allProjectFeatures.filter((feature) => {
        const props = feature.properties || {};
        return this.normalizeProjectId(props.parent_id ?? props.id) === id;
      });
    },

    getLngLatBoundsFromFeatures(features = []) {
      const bounds = {
        minLng: Infinity,
        minLat: Infinity,
        maxLng: -Infinity,
        maxLat: -Infinity,
      };
      let hasCoordinates = false;

      features.forEach((feature) => {
        if (!feature || !feature.geometry) return;
        this.eachCoordinate(feature.geometry.coordinates, (coord) => {
          const lng = Number(coord[0]);
          const lat = Number(coord[1]);
          if (!Number.isFinite(lng) || !Number.isFinite(lat)) return;
          bounds.minLng = Math.min(bounds.minLng, lng);
          bounds.minLat = Math.min(bounds.minLat, lat);
          bounds.maxLng = Math.max(bounds.maxLng, lng);
          bounds.maxLat = Math.max(bounds.maxLat, lat);
          hasCoordinates = true;
        });
      });

      return hasCoordinates ? bounds : null;
    },

    expandLngLatBounds(bounds, meters = 200) {
      if (!bounds) return null;
      const centerLat = (Number(bounds.minLat) + Number(bounds.maxLat)) / 2;
      const latDegrees = Number(meters) / 111320;
      const lngDegrees =
        Number(meters) /
        (111320 * Math.max(0.2, Math.cos((centerLat * Math.PI) / 180)));

      return {
        minLng: Number(bounds.minLng) - lngDegrees,
        minLat: Number(bounds.minLat) - latDegrees,
        maxLng: Number(bounds.maxLng) + lngDegrees,
        maxLat: Number(bounds.maxLat) + latDegrees,
      };
    },

    getBoundsRing(bounds) {
      if (!bounds) return [];
      return [
        [bounds.minLng, bounds.maxLat],
        [bounds.maxLng, bounds.maxLat],
        [bounds.maxLng, bounds.minLat],
        [bounds.minLng, bounds.minLat],
        [bounds.minLng, bounds.maxLat],
      ];
    },

    getCurrentConceptBounds(meters = QADISIYAH_3_CONCEPT_MASK_METERS) {
      const projectId = this.currentProjectId || QADISIYAH_3_PROJECT_ID;
      const features = this.getProjectFeaturesForId(projectId);
      const rawBounds = this.getLngLatBoundsFromFeatures(features);
      return this.expandLngLatBounds(rawBounds, meters);
    },

    getConceptMaskData() {
      const maskBounds = this.getCurrentConceptBounds(
        QADISIYAH_3_CONCEPT_MASK_METERS,
      );
      const innerRing = this.getBoundsRing(maskBounds);
      const worldRing = [
        [-180, -85],
        [180, -85],
        [180, 85],
        [-180, 85],
        [-180, -85],
      ];

      if (!innerRing.length) return EMPTY_FEATURE_COLLECTION;

      return {
        type: "FeatureCollection",
        features: [
          {
            type: "Feature",
            geometry: {
              type: "Polygon",
              coordinates: [worldRing, innerRing],
            },
            properties: { kind: "mask" },
          },
          {
            type: "Feature",
            geometry: {
              type: "LineString",
              coordinates: innerRing,
            },
            properties: { kind: "fade" },
          },
        ],
      };
    },

    applyConceptMapBounds() {
      if (!this.$map || typeof this.$map.setMaxBounds !== "function") return;

      const cameraBounds = this.getCurrentConceptBounds(
        QADISIYAH_3_CONCEPT_CAMERA_METERS,
      );
      if (!cameraBounds) return;

      try {
        this.$map.setMaxBounds([
          [cameraBounds.minLng, cameraBounds.minLat],
          [cameraBounds.maxLng, cameraBounds.maxLat],
        ]);
        this.conceptMapBoundsActive = true;
      } catch (error) {
        console.warn("3D Concept map bounds could not be applied:", error);
      }
    },

    restoreDefaultMapBounds() {
      if (!this.$map || typeof this.$map.setMaxBounds !== "function") return;

      if (!this.conceptMapBoundsActive) return;

      try {
        this.$map.setMaxBounds(MAPBOX_DEFAULT_MAX_BOUNDS);
      } catch (error) {
        console.warn("Default map bounds could not be restored:", error);
      }

      this.conceptMapBoundsActive = false;
    },

    async syncConceptMapLayers() {
      if (!this.$map || !this.$map.isStyleLoaded()) return;

      if (!this.isConceptMapMode() || !this.isQadisiyah3ProjectActive()) {
        this.removeConceptMapLayers();
        return;
      }

      const selectedLayer = this.getSelectedConceptLayer();
      const selectedLayerId = selectedLayer?.id || "";
      const rawImageUrl = this.getConceptLayerImagePath(selectedLayer);
      if (!rawImageUrl) return;

      const imageUrl = await this.getCroppedConceptImageUrl(
        rawImageUrl,
        selectedLayer,
      );
      if (
        !this.isConceptMapMode() ||
        !this.isQadisiyah3ProjectActive() ||
        selectedLayerId !== (this.getSelectedConceptLayer()?.id || "")
      ) {
        return;
      }

      this.applyConceptMapBounds();

      const imageCoordinates = QADISIYAH_3_CONCEPT_IMAGE_COORDINATES;
      try {
        const existingImageSource = this.$map.getSource(
          QADISIYAH_3_CONCEPT_IMAGE_SOURCE_ID,
        );
        if (!existingImageSource) {
          this.$map.addSource(QADISIYAH_3_CONCEPT_IMAGE_SOURCE_ID, {
            type: "image",
            url: imageUrl,
            coordinates: imageCoordinates,
          });
        } else if (typeof existingImageSource.updateImage === "function") {
          existingImageSource.updateImage({
            url: imageUrl,
            coordinates: imageCoordinates,
          });
        }

        if (!this.$map.getLayer(QADISIYAH_3_CONCEPT_IMAGE_LAYER_ID)) {
          this.$map.addLayer({
            id: QADISIYAH_3_CONCEPT_IMAGE_LAYER_ID,
            type: "raster",
            source: QADISIYAH_3_CONCEPT_IMAGE_SOURCE_ID,
            slot: "top",
            paint: {
              "raster-opacity": QADISIYAH_3_CONCEPT_IMAGE_OPACITY,
              "raster-fade-duration": 120,
            },
          });
        }

        const maskData = this.getConceptMaskData();
        const existingMaskSource = this.$map.getSource(
          QADISIYAH_3_CONCEPT_MASK_SOURCE_ID,
        );
        if (!existingMaskSource) {
          this.$map.addSource(QADISIYAH_3_CONCEPT_MASK_SOURCE_ID, {
            type: "geojson",
            data: maskData,
          });
        } else if (typeof existingMaskSource.setData === "function") {
          existingMaskSource.setData(maskData);
        }

        if (!this.$map.getLayer(QADISIYAH_3_CONCEPT_MASK_LAYER_ID)) {
          this.$map.addLayer({
            id: QADISIYAH_3_CONCEPT_MASK_LAYER_ID,
            type: "fill",
            source: QADISIYAH_3_CONCEPT_MASK_SOURCE_ID,
            slot: "top",
            filter: ["==", ["get", "kind"], "mask"],
            paint: {
              "fill-color": "#000000",
              "fill-opacity": 0.78,
            },
          });
        }

        if (!this.$map.getLayer(QADISIYAH_3_CONCEPT_FADE_LAYER_ID)) {
          this.$map.addLayer({
            id: QADISIYAH_3_CONCEPT_FADE_LAYER_ID,
            type: "line",
            source: QADISIYAH_3_CONCEPT_MASK_SOURCE_ID,
            slot: "top",
            filter: ["==", ["get", "kind"], "fade"],
            layout: {
              "line-cap": "round",
              "line-join": "round",
            },
            paint: {
              "line-color": "#000000",
              "line-opacity": 0.62,
              "line-width": 78,
              "line-blur": 32,
            },
          });
        }

        if (typeof this.$map.triggerRepaint === "function") {
          this.$map.triggerRepaint();
        }
      } catch (error) {
        console.warn("3D Concept layers could not be added:", error);
      }
    },

    removeConceptMapLayers() {
      if (!this.$map) return;

      [
        QADISIYAH_3_CONCEPT_FADE_LAYER_ID,
        QADISIYAH_3_CONCEPT_MASK_LAYER_ID,
        QADISIYAH_3_CONCEPT_IMAGE_LAYER_ID,
      ].forEach((layerId) => {
        try {
          if (this.$map && this.$map.getLayer(layerId))
            this.$map.removeLayer(layerId);
        } catch (error) {
          // Layer may have already been removed by a style switch.
        }
      });

      [
        QADISIYAH_3_CONCEPT_MASK_SOURCE_ID,
        QADISIYAH_3_CONCEPT_IMAGE_SOURCE_ID,
      ].forEach((sourceId) => {
        try {
          if (this.$map && this.$map.getSource(sourceId))
            this.$map.removeSource(sourceId);
        } catch (error) {
          // Source may have already been removed by a style switch.
        }
      });

      this.restoreDefaultMapBounds();
    },

    applyConceptMapCamera(duration = 450) {
      if (!this.$map || !this.isQadisiyah3ProjectActive()) return;

      const projectId = this.currentProjectId || QADISIYAH_3_PROJECT_ID;
      const features = this.getProjectFeaturesForId(projectId);
      if (!features.length) return;

      const bounds = new this.mapboxgl.LngLatBounds();
      let hasBounds = false;
      features.forEach((feature) => {
        if (!feature || !feature.geometry) return;
        this.eachCoordinate(feature.geometry.coordinates, (coord) => {
          if (
            Array.isArray(coord) &&
            coord.length >= 2 &&
            Number.isFinite(Number(coord[0])) &&
            Number.isFinite(Number(coord[1]))
          ) {
            bounds.extend([Number(coord[0]), Number(coord[1])]);
            hasBounds = true;
          }
        });
      });

      if (!hasBounds) return;

      this.$map.fitBounds(bounds, {
        padding: { top: 90, right: 70, bottom: 70, left: 70 },
        maxZoom: 16.2,
        pitch: MAPBOX_3D_TERRAIN_PITCH,
        bearing: MAPBOX_3D_TERRAIN_BEARING,
        duration,
      });
    },

    isArabicUI() {
      const lang = document.documentElement.lang || "";
      const dir = document.documentElement.dir || "";
      return lang.startsWith("ar") || dir === "rtl";
    },

    normalizeMapText(value) {
      return String(value || "")
        .trim()
        .toLowerCase()
        .replace(/&/g, "and")
        .replace(/[\-_]/g, " ")
        .replace(/\s+/g, " ");
    },

    isTseTypology(typology) {
      const text = this.normalizeMapText(typology);
      return (
        text === "tse" ||
        text === "tse network" ||
        text === "tse network and water" ||
        text === "tse facilities" ||
        text.includes("tse")
      );
    },

    getTseLotValue(item = {}) {
      const candidates = [
        item.lot,
        item.Lot,
        item.lot_no,
        item.lotNo,
        item.lot_number,
        item.lotNumber,
        item.package,
        item.Package,
        item.package_no,
        item.packageNo,
      ];

      for (const candidate of candidates) {
        const match = String(candidate ?? "").match(/[1-5]/);
        if (match) return Number(match[0]);
      }

      return null;
    },

    parseGeometryField(rawGeometry) {
      if (!rawGeometry) return null;

      let geometry = rawGeometry;

      if (typeof geometry === "string") {
        try {
          geometry = JSON.parse(geometry);
        } catch (e) {
          return null;
        }
      }

      if (geometry?.type === "Feature" && geometry.geometry) {
        geometry = geometry.geometry;
      }

      if (!geometry || typeof geometry !== "object") return null;
      if (!Array.isArray(geometry.coordinates)) return null;

      const expectedDepthByType = {
        Point: 1,
        MultiPoint: 2,
        LineString: 2,
        MultiLineString: 3,
        Polygon: 3,
        MultiPolygon: 4,
      };

      const coordDepth = (coords) => {
        let depth = 0;
        let current = coords;
        while (Array.isArray(current)) {
          depth += 1;
          current = current[0];
        }
        return depth;
      };

      const expectedDepth = expectedDepthByType[geometry.type];
      if (!expectedDepth) return geometry;

      let coords = geometry.coordinates;
      while (
        coordDepth(coords) > expectedDepth &&
        Array.isArray(coords) &&
        coords.length === 1
      ) {
        coords = coords[0];
      }

      return { ...geometry, coordinates: coords };
    },

    parseTseUidList(value) {
      if (value == null) return [];

      if (Array.isArray(value)) {
        return value
          .flatMap((item) => this.parseTseUidList(item))
          .filter(Boolean);
      }

      return String(value)
        .replace(/[\r\n;|]+/g, ",")
        .split(",")
        .map((uid) => uid.trim())
        .filter((uid) => uid && uid.toLowerCase() !== "null");
    },

    getTseGeometryFromRecord(record) {
      if (!record || typeof record !== "object") return null;

      return (
        record.Geometry ??
        record.geometry ??
        record.GEOMETRY ??
        record.geom ??
        record.Shape ??
        record.shape ??
        null
      );
    },

    getTseUidFromRecord(record) {
      if (!record || typeof record !== "object") return "";

      return String(
        record.UID ?? record.uid ?? record.Uid ?? record.id ?? record.ID ?? "",
      ).trim();
    },

    async loadTseGeometryIndex() {
      if (this.tseGeometryByUid) return this.tseGeometryByUid;

      const index = {};

      try {
        const rows = tseGeometryRows;
        const records = Array.isArray(rows)
          ? rows
          : Array.isArray(rows?.features)
            ? rows.features
            : [];

        records.forEach((record) => {
          const uid = this.getTseUidFromRecord(record);
          const rawGeometry = this.getTseGeometryFromRecord(record);
          const geometry = this.parseGeometryField(rawGeometry);

          if (uid && geometry) {
            index[uid] = geometry;
          }
        });
      } catch (error) {
        console.warn(
          "TSE geometry lookup file was not loaded from src/data/tse.json:",
          error,
        );
      }

      this.tseGeometryByUid = index;
      return index;
    },

    addTseReferencedGeometries(
      features,
      item,
      baseProperties,
      columnName,
      geometryRole,
    ) {
      if (!this.isTseTypology(item?.typology_en)) return;
      if (!this.tseGeometryByUid) return;

      const uidList = this.parseTseUidList(item?.[columnName]);
      if (!uidList.length) return;

      const style = this.getTseLineStyle(item, geometryRole);

      uidList.forEach((uid, index) => {
        const geometry = this.tseGeometryByUid[uid];
        if (!geometry) {
          console.warn(`TSE UID not found in src/data/tse.json: ${uid}`);
          return;
        }

        this.pushProjectFeature(
          features,
          geometry,
          {
            ...baseProperties,
            id: `${item.id}-${columnName}-${uid}-${index}`,
            parent_id: item.id,
            tse_geometry_role: geometryRole,
            tse_source_uid: uid,
            tse_lot: this.getTseLotValue(item),
          },
          style,
        );
      });
    },

    getTseLineStyle(item, geometryRole = "primary") {
      const progress = this.normalizeProgressValue(item?.progress_actual);
      const isCompleted = progress >= 100;
      const isSecondaryLike =
        geometryRole === "secondary" || geometryRole === "facilities";

      return {
        strokeColor: isCompleted ? "#47944e" : "#1f6fd1",
        strokeOpacity: isSecondaryLike ? 0.82 : 0.92,
        strokeWeight: isSecondaryLike ? 2 : 5,
        fillColor: "#47944e",
        fillOpacity: 0,
      };
    },

    getBaseProjectStyle(item = {}, geometryRole = "primary") {
      let fillColor = "#61d76c";
      let strokeColor = "#47944e";
      let strokeOpacity = 1;
      let fillOpacity = 0.35;
      let strokeWeight = 1;
      let zIndex = 5;
      let cursor = "pointer";

      if (item.typology_en === "Main Roads") {
        fillColor = "#61d76c";
        strokeColor = "#61d76c";
        fillOpacity = 0;
        strokeWeight = 1;
        zIndex = 9999;
      } else if (
        item.typology_en === "Pocket Parks" ||
        item.typology_en === "Neighborhood Parks"
      ) {
        fillColor = "#61d76c";
        strokeColor = "#47944e";
        strokeOpacity = 0;
        fillOpacity = 0.35;
        strokeWeight = 0;
        cursor = "default";
        zIndex = 1;
      }

      const progress = this.normalizeProgressValue(item.progress_actual);
      const inProgress = progress >= 0 && progress < 100;

      if (this.isTseTypology(item.typology_en)) {
        return {
          ...this.getTseLineStyle(item, geometryRole),
          zIndex,
          cursor,
        };
      }

      if (inProgress) {
        strokeColor = "#ffae00";
        fillColor = item.typology_en === "Main Roads" ? "#ffae00" : "#ffae00";
        if (fillOpacity === 0 && item.typology_en !== "Main Roads") {
          fillOpacity = 0.35;
        }
      }

      return {
        fillColor,
        strokeColor,
        strokeOpacity,
        fillOpacity,
        strokeWeight,
        zIndex,
        cursor,
      };
    },

    getTriangleSvg(fillColor, strokeColor = "#ffd700") {
      return `
        <svg xmlns="http://www.w3.org/2000/svg" width="32" height="32" viewBox="0 0 32 32">
          <path d="M16 3 L29 28 L3 28 Z" fill="${fillColor}" fill-opacity="0.95" stroke="${strokeColor}" stroke-width="2"/>
        </svg>
      `;
    },

    addMapboxImage(name, svg) {
      if (!this.$map || this.$map.hasImage(name)) return Promise.resolve();

      return new Promise((resolve, reject) => {
        const image = new Image(32, 32);
        image.onload = () => {
          if (this.$map && !this.$map.hasImage(name)) {
            this.$map.addImage(name, image);
          }
          resolve();
        };
        image.onerror = reject;
        image.src = `data:image/svg+xml;charset=utf-8,${encodeURIComponent(svg)}`;
      });
    },

    async addMapboxImages() {
      await Promise.all([
        this.addMapboxImage(
          "facility-triangle-completed",
          this.getTriangleSvg("#47944e"),
        ),
        this.addMapboxImage(
          "facility-triangle-progress",
          this.getTriangleSvg("#ffd700"),
        ),
        this.addMapboxImage(
          "facility-triangle-highlighted",
          this.getTriangleSvg("#fff36a"),
        ),
      ]);
    },

    async addProjectLayers() {
      if (!this.$map || !this.$map.getStyle()) return;

      await this.addMapboxImages();

      if (!this.$map.getSource(PROJECT_SOURCE_ID)) {
        this.$map.addSource(PROJECT_SOURCE_ID, {
          type: "geojson",
          data: EMPTY_FEATURE_COLLECTION,
        });
      }

      if (!this.$map.getLayer(PROJECT_FILL_LAYER_ID)) {
        this.$map.addLayer({
          id: PROJECT_FILL_LAYER_ID,
          type: "fill",
          source: PROJECT_SOURCE_ID,
          slot: "top",
          filter: [
            "all",
            ["==", ["get", "visible"], true],
            [
              "any",
              ["==", ["geometry-type"], "Polygon"],
              ["==", ["geometry-type"], "MultiPolygon"],
            ],
          ],
          paint: {
            "fill-color": ["get", "fillColor"],
            "fill-opacity": ["to-number", ["get", "fillOpacity"], 0.35],
          },
        });
      }

      if (!this.$map.getLayer(PROJECT_LINE_LAYER_ID)) {
        this.$map.addLayer({
          id: PROJECT_LINE_LAYER_ID,
          type: "line",
          source: PROJECT_SOURCE_ID,
          slot: "top",
          filter: [
            "all",
            ["==", ["get", "visible"], true],
            [
              "any",
              ["==", ["geometry-type"], "Polygon"],
              ["==", ["geometry-type"], "MultiPolygon"],
              ["==", ["geometry-type"], "LineString"],
              ["==", ["geometry-type"], "MultiLineString"],
            ],
          ],
          layout: {
            "line-cap": "round",
            "line-join": "round",
            // Shared TSE UIDs create overlapping line features for multiple
            // projects. Keep the currently highlighted project above the
            // non-highlighted copy so either project can be highlighted.
            "line-sort-key": ["to-number", ["get", "renderSortKey"], 0],
          },
          paint: {
            "line-color": ["get", "strokeColor"],
            "line-opacity": ["to-number", ["get", "strokeOpacity"], 1],
            "line-width": ["to-number", ["get", "strokeWeight"], 1],
          },
        });
      }

      if (!this.$map.getLayer(PROJECT_FACILITY_LAYER_ID)) {
        this.$map.addLayer({
          id: PROJECT_FACILITY_LAYER_ID,
          type: "symbol",
          source: PROJECT_SOURCE_ID,
          slot: "top",
          filter: [
            "all",
            ["==", ["get", "visible"], true],
            ["==", ["geometry-type"], "Point"],
            ["==", ["get", "tse_geometry_role"], "facilities"],
          ],
          layout: {
            "icon-image": [
              "coalesce",
              ["get", "facilityIcon"],
              "facility-triangle-progress",
            ],
            "icon-size": 0.82,
            "icon-allow-overlap": true,
            "icon-ignore-placement": true,
          },
        });
      }

      this.projectLayersReady = true;
      this.bindProjectLayerEvents();
      this.renderMapData();
    },

    bindProjectLayerEvents() {
      if (!this.$map || this.mapLayerEventsBound) return;

      const layerIds = [
        PROJECT_FILL_LAYER_ID,
        PROJECT_LINE_LAYER_ID,
        PROJECT_FACILITY_LAYER_ID,
      ];

      layerIds.forEach((layerId) => {
        this.$map.on("mouseenter", layerId, this.onProjectMouseEnter);
        this.$map.on("mousemove", layerId, this.onProjectMouseMove);
        this.$map.on("mouseleave", layerId, this.onProjectMouseLeave);
        this.$map.on("click", layerId, this.onProjectClick);
      });

      this.mapLayerEventsBound = true;
    },

    getProperty(feature, key) {
      if (!feature) return undefined;
      if (typeof feature.getProperty === "function")
        return feature.getProperty(key);
      if (
        feature.properties &&
        Object.prototype.hasOwnProperty.call(feature.properties, key)
      ) {
        return feature.properties[key];
      }
      return feature[key];
    },

    getFeatureProjectId(feature) {
      return this.normalizeProjectId(
        this.getProperty(feature, "parent_id") ??
          this.getProperty(feature, "id"),
      );
    },

    normalizeProjectId(value) {
      return String(value ?? "").trim();
    },

    getMarkerProjectId(marker) {
      return this.normalizeProjectId(
        marker && (marker.parent_id ?? marker.parentId ?? marker.id),
      );
    },

    hasActiveSearchFilter() {
      return Array.isArray(this.activeSearchProjectIds);
    },

    matchesActiveSearchProject(projectId) {
      if (!this.hasActiveSearchFilter()) return true;

      const normalizedId = this.normalizeProjectId(projectId);
      return this.activeSearchProjectIds.includes(normalizedId);
    },

    getLegendStatusKey(progressActual) {
      const progress = this.normalizeProgressValue(progressActual);
      return progress >= 100 ? "completed" : "progress";
    },

    passesLegendStatusFilter(progressActual) {
      const statusKey = this.getLegendStatusKey(progressActual);
      return this.legendStatusFilters[statusKey] !== false;
    },

    passesContractMapFilter(progressActual) {
      const progress = this.normalizeProgressValue(progressActual);

      if (
        this.contractMapFilter === "closed" ||
        this.contractMapFilter === "completed"
      ) {
        return progress >= 100;
      }
      if (
        this.contractMapFilter === "open" ||
        this.contractMapFilter === "in-progress" ||
        this.contractMapFilter === "progress"
      ) {
        return progress < 100;
      }
      return true;
    },

    toggleLegendStatusFilter(statusKey) {
      if (
        !Object.prototype.hasOwnProperty.call(
          this.legendStatusFilters,
          statusKey,
        )
      ) {
        return;
      }

      this.legendStatusFilters = {
        ...this.legendStatusFilters,
        [statusKey]: !this.legendStatusFilters[statusKey],
      };

      this.refreshMapForCurrentFilters();
    },

    updateTseLegendVisibility(show = false, mode = "default") {
      this.tseLegendVisible = show;
      this.currentLegendMode = mode;
    },

    refreshMapForCurrentFilters() {
      this.renderMapData();
    },

    fitMapToActiveSearchResults() {
      if (
        !this.$map ||
        !this.hasActiveSearchFilter() ||
        !this.activeSearchProjectIds.length
      ) {
        return;
      }

      this.fitMapToProjectIds(this.activeSearchProjectIds, 13);
    },

    async showOnlySearchResults(projectIds = []) {
      await this.waitForMapReady();

      this.activeSearchProjectIds = Array.from(
        new Set(
          (Array.isArray(projectIds) ? projectIds : [])
            .map((id) => this.normalizeProjectId(id))
            .filter(Boolean),
        ),
      );
      this.activeViewMode = "all";
      this.currentProjectId = null;
      this.removeThreeParkLayer();
      this.ensureConceptModeAllowed();
      this.renderMapData();

      window.setTimeout(() => {
        this.fitMapToActiveSearchResults();
      }, 100);
    },

    categoryMatchesTypology(category, typology) {
      const categoryText = this.normalizeMapText(category || "all");
      const typologyText = this.normalizeMapText(typology);

      if (categoryText === "all" || categoryText === "") {
        return !this.isTseTypology(typologyText);
      }

      if (categoryText.includes("large park") || categoryText === "parks") {
        return (
          typologyText.includes("large park") ||
          typologyText.includes("pocket park") ||
          typologyText.includes("neighborhood park") ||
          typologyText === "parks"
        );
      }

      if (
        categoryText.includes("natural park") ||
        categoryText.includes("wadi") ||
        categoryText.includes("valley")
      ) {
        return (
          typologyText.includes("natural park") ||
          typologyText.includes("wadi") ||
          typologyText.includes("valley") ||
          typologyText.includes("tributar")
        );
      }

      if (
        categoryText.includes("public complex") ||
        categoryText === "complex" ||
        categoryText === "complexes"
      ) {
        return (
          typologyText.includes("public complex") ||
          typologyText.includes("complex") ||
          typologyText.includes("public facilities") ||
          typologyText.includes("public facility") ||
          typologyText.includes("public building")
        );
      }

      if (categoryText.includes("nurser")) {
        return (
          typologyText.includes("nurser") || typologyText.includes("trees")
        );
      }

      if (this.isTseTypology(categoryText)) {
        return this.isTseTypology(typologyText);
      }

      return categoryText === typologyText;
    },

    markerMatchesCategory(marker, category) {
      return this.categoryMatchesTypology(category, marker.typology_en);
    },

    featureMatchesCategory(feature, category) {
      return this.categoryMatchesTypology(
        category,
        this.getProperty(feature, "typology_en"),
      );
    },

    applyContractMapFilterToMarkers() {
      // Mapbox version draws your geometries as GeoJSON layers, so the old
      // Google Maps Marker array is kept only for backwards compatibility.
    },

    applyContractMapFilterToShapes() {
      this.renderMapData();
    },

    parseContractDateValue(value) {
      if (value === null || value === undefined || value === "") return null;

      if (value instanceof Date) {
        const date = new Date(value.getTime());
        if (Number.isNaN(date.getTime())) return null;
        date.setHours(0, 0, 0, 0);
        return date;
      }

      const text = String(value).trim();
      if (!text || text.toLowerCase() === "null") return null;

      const ymdMatch = text.match(/^(\d{4})[-/](\d{1,2})[-/](\d{1,2})/);
      if (ymdMatch) {
        const [, year, month, day] = ymdMatch;
        const date = new Date(Number(year), Number(month) - 1, Number(day));
        if (!Number.isNaN(date.getTime())) {
          date.setHours(0, 0, 0, 0);
          return date;
        }
      }

      const dmyMatch = text.match(/^(\d{1,2})[-/](\d{1,2})[-/](\d{4})/);
      if (dmyMatch) {
        const [, day, month, year] = dmyMatch;
        const date = new Date(Number(year), Number(month) - 1, Number(day));
        if (!Number.isNaN(date.getTime())) {
          date.setHours(0, 0, 0, 0);
          return date;
        }
      }

      const serialNumber = Number(text);
      if (
        Number.isFinite(serialNumber) &&
        serialNumber > 20000 &&
        serialNumber < 80000
      ) {
        const date = new Date(
          Math.round((serialNumber - 25569) * 86400 * 1000),
        );
        if (!Number.isNaN(date.getTime())) {
          date.setHours(0, 0, 0, 0);
          return date;
        }
      }

      const parsed = new Date(text);
      if (Number.isNaN(parsed.getTime())) return null;
      parsed.setHours(0, 0, 0, 0);
      return parsed;
    },

    getLatestContractEndDisplay(completionDate, contractExpireDate) {
      const completionParsed = this.parseContractDateValue(completionDate);
      const contractExpireParsed =
        this.parseContractDateValue(contractExpireDate);

      if (completionParsed && contractExpireParsed) {
        return contractExpireParsed.getTime() > completionParsed.getTime()
          ? contractExpireDate
          : completionDate;
      }

      if (contractExpireParsed) return contractExpireDate;
      if (completionParsed) return completionDate;
      return null;
    },

    getFeatureContractEndDisplay(feature) {
      const completionDate =
        this.getProperty(feature, "completion_date") ||
        this.getProperty(feature, "completionDate") ||
        this.getProperty(feature, "completion_date_en");
      const contractExpireDate =
        this.getProperty(feature, "Contract_Expire") ||
        this.getProperty(feature, "contract_expire") ||
        this.getProperty(feature, "contractExpire") ||
        this.getProperty(feature, "ContractExpire");

      return (
        this.getLatestContractEndDisplay(completionDate, contractExpireDate) ||
        this.getProperty(feature, "contract_end_date") ||
        null
      );
    },

    formatNumberWithCommas(value) {
      if (value === null || value === undefined || value === "") return "0";
      const num = Number(String(value).replace(/,/g, "").trim());
      if (!Number.isNaN(num)) return num.toLocaleString();
      return String(value);
    },

    formatTooltipTrees(item) {
      const actual = Number(item?.trees_planted ?? 0);
      const target = Number(item?.trees_target ?? 0);
      const actualText = this.formatNumberWithCommas(actual);
      if (target && target !== actual) {
        return `${actualText} (of ${this.formatNumberWithCommas(target)})`;
      }
      return actualText;
    },

    drawLineString(coords) {
      if (!this.$map || !Array.isArray(coords)) return null;
      const sourceId = `temporary-line-${Date.now()}`;
      const layerId = `${sourceId}-layer`;
      const data = {
        type: "FeatureCollection",
        features: [
          {
            type: "Feature",
            geometry: {
              type: "LineString",
              coordinates: coords,
            },
            properties: {},
          },
        ],
      };
      this.$map.addSource(sourceId, { type: "geojson", data });
      this.$map.addLayer({
        id: layerId,
        type: "line",
        source: sourceId,
        paint: {
          "line-color": "#0D47A1",
          "line-width": 5,
          "line-opacity": 1,
        },
      });
      return { sourceId, layerId };
    },

    switch_markers_icons(type) {
      this.icons.active.icon = "/images/marker-selected.gif";
      this.icons.inactive.icon = "/images/marker-unselected-" + type + "2.png";
      this.renderMapData();
    },

    set_map_dark(type) {
      if (!this.$map) return;

      // Keep Satellite and 3D modes in their own Mapbox styles. Dark mode only
      // changes the normal Street basemap.
      if (this.activeBaseMapMode !== "street") {
        return;
      }

      this.projectLayersReady = false;
      this.unbindProjectLayerEvents();
      this.removeThreeParkLayer();
      this.$map.setStyle(this.getMapboxStyleForMode("street", type));
      this.$map.once("idle", () => {
        this.restoreMapOverlaysAfterStyleChange({ moveCamera: false });
      });
    },

    async load_near_projects(itemOrDistrict, districtOrAction, action = true) {
      const mapReady = await this.waitForMapReady();
      if (!mapReady) return;

      const district =
        districtOrAction && typeof districtOrAction === "object"
          ? districtOrAction
          : itemOrDistrict;
      const project =
        itemOrDistrict && itemOrDistrict.id ? itemOrDistrict : district;
      const centerValue = district?.center || project?.center;
      const center = this.getLngLatFromCenterValue(centerValue);

      this.nearProjectId = this.normalizeProjectId(project?.id || district?.id);
      this.nearProjectCenter = center;
      this.activeViewMode = "near";
      this.removeThreeParkLayer();
      this.ensureConceptModeAllowed();
      this.renderMapData();

      if (center) {
        this.$map.easeTo({
          center,
          zoom: 12,
          duration: action === false ? 0 : 300,
        });
      }
    },

    async load_category(category, backRouter) {
      await this.load_markers(category, backRouter);
    },

    async load_markers(category = null, backRouter) {
      await this.waitForMapReady();

      if (!this.$map) return;

      this.activeViewMode = "all";
      this.currentProjectId = null;
      this.nearProjectId = null;
      this.nearProjectCenter = null;
      this.removeThreeParkLayer();
      this.ensureConceptModeAllowed();

      const isSearchPage =
        Array.isArray(backRouter) &&
        String(backRouter[0] || "").toLowerCase() === "search";

      if (!isSearchPage) {
        this.activeSearchProjectIds = null;
      }

      this.currentMarkerCategory = category;
      this.currentBackRouter = backRouter;
      this.mainMapView = this.getMainMapViewByCategory(category);

      if (!this.allProjectFeatures.length) {
        await this.load_all_map();
      }

      this.updateTseLegendVisibility(
        category !== null,
        this.isTseTypology(category) ? "tse" : "default",
      );
      this.renderMapData();
      this.pan_to(this.latitude, this.longitude, this.zoom);
    },

    load_project_marker(item, index, toggle) {
      if (!item || item.id == null) {
        this.highlightedProjectId = null;
      } else {
        this.highlightedProjectId = toggle
          ? this.normalizeProjectId(item.id)
          : null;
      }
      this.renderMapData();
    },

    async load_districts(category, backRouter) {
      await this.waitForMapReady();
      if (!this.$map) return;

      const doc = this.$root.districts_src || [];
      const features = [];

      doc.forEach((district, index) => {
        let shape = Array.isArray(district.shape)
          ? district.shape.join(" ")
          : district.shape;
        if (!shape) return;

        const ring = String(shape)
          .split(" ")
          .map((pair) => pair.split(",").map(Number))
          .filter((pair) => pair.length === 2 && pair.every(Number.isFinite));

        if (ring.length < 3) return;
        const first = ring[0];
        const last = ring[ring.length - 1];
        if (first[0] !== last[0] || first[1] !== last[1]) {
          ring.push(first);
        }

        features.push({
          type: "Feature",
          id: `district-${index}`,
          geometry: {
            type: "Polygon",
            coordinates: [ring],
          },
          properties: {
            id: district.id ?? index,
            indexID: index,
            category,
            backRouter0: Array.isArray(backRouter) ? backRouter[0] : "home",
            backRouter1: Array.isArray(backRouter) ? backRouter[1] : "Overview",
          },
        });
      });

      this.districtFeatures = features;
      this.ensureDistrictLayers();
      this.setDistrictSourceData();
      this.fitFeatures(features, 13);
    },

    ensureDistrictLayers() {
      if (!this.$map || !this.$map.getStyle()) return;

      if (!this.$map.getSource(DISTRICT_SOURCE_ID)) {
        this.$map.addSource(DISTRICT_SOURCE_ID, {
          type: "geojson",
          data: EMPTY_FEATURE_COLLECTION,
        });
      }

      if (!this.$map.getLayer(DISTRICT_FILL_LAYER_ID)) {
        this.$map.addLayer({
          id: DISTRICT_FILL_LAYER_ID,
          type: "fill",
          source: DISTRICT_SOURCE_ID,
          slot: "top",
          paint: {
            "fill-color": "#55e0e5",
            "fill-opacity": 0.35,
          },
        });
      }

      if (!this.$map.getLayer(DISTRICT_LINE_LAYER_ID)) {
        this.$map.addLayer({
          id: DISTRICT_LINE_LAYER_ID,
          type: "line",
          source: DISTRICT_SOURCE_ID,
          slot: "top",
          paint: {
            "line-color": "#3f84ee",
            "line-width": 1,
            "line-opacity": 1,
          },
        });

        this.$map.on("click", DISTRICT_FILL_LAYER_ID, (event) => {
          const feature = event.features && event.features[0];
          if (!feature) return;
          this.$router.push(
            "/district_projects/" +
              feature.properties.indexID +
              "/" +
              feature.properties.backRouter0 +
              "/" +
              feature.properties.backRouter1,
          );
        });
      }
    },

    setDistrictSourceData() {
      const source = this.$map && this.$map.getSource(DISTRICT_SOURCE_ID);
      if (!source || !source.setData) return;
      source.setData({
        type: "FeatureCollection",
        features: this.districtFeatures,
      });
    },

    clear_map() {
      this.markers = [];
      this.highlightedProjectId = null;
      this.hoveredFeatureKey = null;
      this.updateTseLegendVisibility(false);
      this.districtFeatures = [];
      if (this.$map && this.$map.getSource(DISTRICT_SOURCE_ID)) {
        this.setDistrictSourceData();
      }
      this.renderMapData();
    },

    async load_project(item) {
      const mapReady = await this.waitForMapReady();
      if (!mapReady || !item) return;

      if (!this.allProjectFeatures.length) {
        await this.load_all_map();
      }

      this.activeViewMode = "project";
      this.currentProjectId = this.normalizeProjectId(item.id);
      this.resetStandaloneThreeLayerFilters();
      this.activeSearchProjectIds = null;
      this.highlightedProjectId = null;
      this.ensureConceptModeAllowed();
      this.updateTseLegendVisibility(false);
      this.renderMapData();
      this.rebuildThreeParkLayer();

      window.setTimeout(() => {
        if (this.isStandaloneThreeMode) {
          this.resetThreeParkCamera();
        } else {
          this.fitMapToProjectIds([this.currentProjectId], 14);
        }
      }, 100);
    },

    calc_distance(mk1, mk2) {
      const getLat = (point) =>
        Array.isArray(point)
          ? Number(point[1])
          : Number(point.lat ?? point.latitude);
      const getLng = (point) =>
        Array.isArray(point)
          ? Number(point[0])
          : Number(point.lng ?? point.longitude);

      const lat1 = getLat(mk1);
      const lng1 = getLng(mk1);
      const lat2 = getLat(mk2);
      const lng2 = getLng(mk2);
      if (![lat1, lng1, lat2, lng2].every(Number.isFinite)) return Infinity;

      const R = 3958.8;
      const rlat1 = lat1 * (Math.PI / 180);
      const rlat2 = lat2 * (Math.PI / 180);
      const difflat = rlat2 - rlat1;
      const difflon = (lng2 - lng1) * (Math.PI / 180);

      return (
        2 *
        R *
        Math.asin(
          Math.sqrt(
            Math.sin(difflat / 2) * Math.sin(difflat / 2) +
              Math.cos(rlat1) *
                Math.cos(rlat2) *
                Math.sin(difflon / 2) *
                Math.sin(difflon / 2),
          ),
        )
      );
    },

    async load_json() {
      await this.load_districts(null, ["home", "Overview"]);
    },

    async init_map() {
      if (!this.mapboxgl) {
        await this.ensureMapboxLoaded();
      }

      if (this.$map) return;

      this.mapReadyPromise = new Promise((resolve) => {
        this.mapReadyResolve = resolve;
      });

      this.$map = new this.mapboxgl.Map({
        container: this.$refs.map || "map",
        style: this.getMapboxStyleForMode(),
        language: this.getMapboxLanguage(),
        // Three.js custom layers require Mapbox's Mercator projection and
        // antialiasing keeps their wireframe edges smoother.
        projection: "mercator",
        antialias: true,
        center: this.getDefaultCenter(),
        zoom: Number(this.zoom || 10),
        minZoom: 1,
        maxZoom: 18,
        attributionControl: true,
        maxBounds: MAPBOX_DEFAULT_MAX_BOUNDS,
      });

      this.mapPopup = new this.mapboxgl.Popup({
        closeButton: false,
        closeOnClick: false,
        offset: 12,
      });

      this.configureMapboxScrollZoomSensitivity();
      this.enableMiddleMousePanNavigation();

      this.$map.on("load", async () => {
        this.configureMapboxScrollZoomSensitivity();
        await this.restoreMapOverlaysAfterStyleChange({ moveCamera: false });
        this.preservedMapViewState = this.captureMapboxViewState();
        await this.load_all_map();
        this.updateTseLegendVisibility(true, "default");
        this.renderMapData();
        if (this.mapReadyResolve) this.mapReadyResolve(true);
      });

      this.$map.on("style.load", this.handleMapStyleLoad);

      this.$map.on("zoomend", () => {
        this.renderMapData();
      });

      this.$map.on("error", (event) => {
        console.warn(
          "Mapbox map warning:",
          event && event.error ? event.error : event,
        );
      });

      await this.mapReadyPromise;
    },

    async load_all_map() {
      await this.loadTseGeometryIndex();

      const doc = this.$root.data_src || [];
      const features = [];

      doc.forEach((item) => {
        if (item.hidden === "yes") return;

        const treesPlanted =
          item.trees_planted ??
          item.treesPlanted ??
          item.trees_planted_count ??
          0;
        const treesTarget =
          item.trees_target ?? item.treesTarget ?? item.trees_target_count ?? 0;
        const progressActual =
          item.progress_actual ?? item.progressActual ?? item.progress ?? null;
        const progressPlanned =
          item.progress_planned ??
          item.progressPlanned ??
          item.planned_progress ??
          null;
        const completionDate =
          item.completion_date ??
          item.completionDate ??
          item.completion_date_en ??
          null;
        const contractExpireDate =
          item.Contract_Expire ??
          item.contract_expire ??
          item.contractExpire ??
          item.ContractExpire ??
          null;
        const contractEndDate = this.getLatestContractEndDisplay(
          completionDate,
          contractExpireDate,
        );

        const baseProperties = {
          id: item.id,
          parent_id: item.id,
          level: Number(item.level ?? 0),
          typology_en: item.typology_en,
          typology_ar: item.typology_ar,
          near_project_center: item.center,
          name_ar: item.name_ar,
          name_en: item.name_en,
          trees_planted: treesPlanted,
          trees_target: treesTarget,
          progress_actual: progressActual,
          progress_planned: progressPlanned,
          completion_date: completionDate,
          Contract_Expire: contractExpireDate,
          contract_expire: contractExpireDate,
          contract_end_date: contractEndDate,
          cursor: "pointer",
        };

        const primaryGeometry = this.parseGeometryField(item.geometry);
        if (primaryGeometry) {
          const primaryStyle = this.getBaseProjectStyle(item, "primary");
          this.pushProjectFeature(
            features,
            primaryGeometry,
            {
              ...baseProperties,
              tse_geometry_role: this.isTseTypology(item.typology_en)
                ? "primary"
                : null,
              tse_lot: this.getTseLotValue(item),
            },
            primaryStyle,
          );
        }

        this.addTseReferencedGeometries(
          features,
          item,
          baseProperties,
          "tse_primary",
          "primary",
        );

        this.addTseReferencedGeometries(
          features,
          item,
          baseProperties,
          "tse_secondary",
          "secondary",
        );

        const secondaryGeometry = this.parseGeometryField(item.geometry_B);
        if (this.isTseTypology(item.typology_en) && secondaryGeometry) {
          this.pushProjectFeature(
            features,
            secondaryGeometry,
            {
              ...baseProperties,
              id: `${item.id}-geometry-b`,
              parent_id: item.id,
              tse_geometry_role: "secondary",
              tse_lot: this.getTseLotValue(item),
            },
            this.getTseLineStyle(item, "secondary"),
          );
        }

        const facilitiesGeometry = this.parseGeometryField(
          item.geometry_Facilities,
        );
        if (this.isTseTypology(item.typology_en) && facilitiesGeometry) {
          this.pushProjectFeature(
            features,
            facilitiesGeometry,
            {
              ...baseProperties,
              id: `${item.id}-geometry-facilities`,
              parent_id: item.id,
              tse_geometry_role: "facilities",
              tse_lot: this.getTseLotValue(item),
            },
            this.getTseLineStyle(item, "facilities"),
          );
        }
      });

      this.allProjectFeatures = features;
      this.renderMapData();
    },

    pushProjectFeature(features, geometry, properties, style) {
      if (!geometry || !properties) return;

      const key = `${properties.parent_id ?? properties.id}-${properties.id}-${properties.tse_geometry_role ?? "geometry"}-${features.length}`;
      const progress = this.normalizeProgressValue(properties.progress_actual);
      const isCompleted = progress >= 100;

      features.push({
        type: "Feature",
        id: key,
        geometry,
        properties: {
          ...properties,
          _key: key,
          baseFillColor: style.fillColor || "#61d76c",
          baseStrokeColor: style.strokeColor || "#47944e",
          baseFillOpacity: Number(style.fillOpacity ?? 0.35),
          baseStrokeOpacity: Number(style.strokeOpacity ?? 1),
          baseStrokeWeight: Number(style.strokeWeight ?? 1),
          zIndex: Number(style.zIndex ?? 5),
          cursor: style.cursor || properties.cursor || "pointer",
          visible: false,
          fillColor: style.fillColor || "#61d76c",
          strokeColor: style.strokeColor || "#47944e",
          fillOpacity: Number(style.fillOpacity ?? 0.35),
          strokeOpacity: Number(style.strokeOpacity ?? 1),
          strokeWeight: Number(style.strokeWeight ?? 1),
          facilityIcon: isCompleted
            ? "facility-triangle-completed"
            : "facility-triangle-progress",
        },
      });
    },

    set_zoom(zoom) {
      if (!this.$map) return;
      this.$map.setZoom(Number(zoom));
    },

    pan_to(latitude, longitude, zoom) {
      if (!this.$map) return;
      const center = [Number(longitude), Number(latitude)];
      if (!center.every(Number.isFinite)) return;
      this.$map.easeTo({ center, zoom: Number(zoom), duration: 250 });
    },

    load_markers_by_zoom() {
      this.renderMapData();
    },

    async load_markers_by_zoom_category(category) {
      await this.load_markers(category, this.currentBackRouter);
    },

    load_markers_by_zoom_id(id) {
      this.currentProjectId = this.normalizeProjectId(id);
      this.resetStandaloneThreeLayerFilters();
      this.activeViewMode = "project";
      this.ensureConceptModeAllowed();
      this.renderMapData();
      this.rebuildThreeParkLayer();
    },

    load_markers_by_zoom_near_projects(id, center) {
      this.nearProjectId = this.normalizeProjectId(id);
      this.nearProjectCenter = this.normalizeLngLat(center);
      this.activeViewMode = "near";
      this.removeThreeParkLayer();
      this.ensureConceptModeAllowed();
      this.renderMapData();
    },

    async load_markers_by_zoom_all() {
      this.activeViewMode = "all";
      this.currentMarkerCategory = "all";
      this.removeThreeParkLayer();
      this.ensureConceptModeAllowed();
      this.renderMapData();
    },

    getLngLatFromCenterValue(centerValue) {
      if (!centerValue) return null;
      if (Array.isArray(centerValue)) {
        if (centerValue.length < 2) return null;
        return [Number(centerValue[1]), Number(centerValue[0])];
      }
      if (Array.isArray(centerValue.coordinates)) {
        return [
          Number(centerValue.coordinates[0]),
          Number(centerValue.coordinates[1]),
        ];
      }
      if (centerValue.lng != null && centerValue.lat != null) {
        return [Number(centerValue.lng), Number(centerValue.lat)];
      }
      if (centerValue.longitude != null && centerValue.latitude != null) {
        return [Number(centerValue.longitude), Number(centerValue.latitude)];
      }
      return null;
    },

    normalizeLngLat(value) {
      if (!value) return null;
      if (Array.isArray(value)) return [Number(value[0]), Number(value[1])];
      if (value.lng != null && value.lat != null)
        return [Number(value.lng), Number(value.lat)];
      if (value.longitude != null && value.latitude != null)
        return [Number(value.longitude), Number(value.latitude)];
      return this.getLngLatFromCenterValue(value);
    },

    shouldFeatureBeVisible(feature) {
      const props = feature.properties || {};
      const zoom = this.$map ? this.$map.getZoom() : Number(this.zoom || 10);
      const projectId = this.normalizeProjectId(props.parent_id ?? props.id);
      const category = this.currentMarkerCategory;
      const level = Number(props.level ?? 0);

      if (this.activeViewMode === "project" && this.currentProjectId) {
        return projectId === this.currentProjectId;
      }

      if (this.activeViewMode === "near" && this.nearProjectCenter) {
        const center = this.getLngLatFromCenterValue(props.near_project_center);
        if (projectId === this.nearProjectId) return true;
        if (!center) return false;
        return this.calc_distance(this.nearProjectCenter, center) < 5;
      }

      const categoryMatch = this.hasActiveSearchFilter()
        ? this.matchesActiveSearchProject(projectId)
        : this.categoryMatchesTypology(category, props.typology_en);

      const statusMatch =
        this.passesContractMapFilter(props.progress_actual) &&
        this.passesLegendStatusFilter(props.progress_actual);

      return categoryMatch && statusMatch && level <= zoom;
    },

    getStyledFeature(feature) {
      const props = { ...(feature.properties || {}) };
      const visible = this.shouldFeatureBeVisible(feature);
      const projectId = this.normalizeProjectId(props.parent_id ?? props.id);
      const progress = this.normalizeProgressValue(props.progress_actual);
      const inProgress = progress >= 0 && progress < 100;
      const isTse = this.isTseTypology(props.typology_en);
      const isHighlighted =
        projectId === this.highlightedProjectId ||
        props._key === this.hoveredFeatureKey;
      const isPocketPark =
        props.typology_en === "Pocket Parks" ||
        props.typology_en === "Neighborhood Parks";

      let fillColor = props.baseFillColor || "#61d76c";
      let strokeColor = props.baseStrokeColor || "#47944e";
      let fillOpacity = Number(props.baseFillOpacity ?? 0.35);
      let strokeOpacity = Number(props.baseStrokeOpacity ?? 1);
      let strokeWeight = Number(props.baseStrokeWeight ?? 1);
      let facilityIcon =
        progress >= 100
          ? "facility-triangle-completed"
          : "facility-triangle-progress";

      if (isTse) {
        const tseStyle = this.getTseLineStyle(
          { progress_actual: props.progress_actual, lot: props.tse_lot },
          props.tse_geometry_role,
        );
        fillColor = tseStyle.fillColor;
        fillOpacity = tseStyle.fillOpacity;
        strokeColor = tseStyle.strokeColor;
        strokeOpacity = tseStyle.strokeOpacity;
        strokeWeight = tseStyle.strokeWeight;
      } else if (props.typology_en === "Green Neighborhoods") {
        if (inProgress) {
          fillColor = "#ffae00";
          strokeColor = "#ffae00";
        } else {
          fillColor = "#61d76c";
          strokeColor = "#47944e";
        }
        fillOpacity = 0.35;
        strokeOpacity = 1;
        strokeWeight = 1;
      } else if (props.typology_en === "Main Roads") {
        fillOpacity = 0;
        strokeColor = inProgress ? "#ffae00" : "#61d76c";
        strokeOpacity = 1;
        strokeWeight = Number(props.baseStrokeWeight ?? 1);
      } else if (inProgress) {
        fillColor = "#ffae00";
        strokeColor = "#ffae00";
        if (fillOpacity === 0) fillOpacity = 0.35;
      }

      if (isHighlighted && !isPocketPark) {
        strokeColor = "#fffb4f";
        strokeOpacity = 1;
        fillColor = isTse ? fillColor : "#fffb4f";
        fillOpacity = isTse ? 0 : 0.65;
        facilityIcon = "facility-triangle-highlighted";

        // Make a highlighted TSE line wider than any identical shared-UID
        // line underneath it. This also makes the highlight easier to see.
        if (isTse) {
          strokeWeight = Math.max(strokeWeight + 3, 7);
        }
      }

      // In 3D mode the terrain/satellite texture needs to remain visible.
      // A strong green/orange fill hides the mountain relief and makes the
      // landscape look flat, so lower normal polygon opacity only in 3D.
      if (this.isThreeDimensionalMapMode() && !isTse && !isHighlighted) {
        fillOpacity = Math.min(fillOpacity, 0.22);
      }

      if (
        this.isConceptMapMode() &&
        projectId === this.normalizeProjectId(this.currentProjectId) &&
        !isTse
      ) {
        fillColor = "#000000";
        strokeColor = "#000000";
        fillOpacity = 0;
        strokeOpacity = 0;
        strokeWeight = 0;
      }

      return {
        ...feature,
        properties: {
          ...props,
          visible,
          fillColor,
          strokeColor,
          fillOpacity,
          strokeOpacity,
          strokeWeight,
          // Mapbox uses this in line-sort-key. A highlighted project receives
          // the highest sort key, so its shared TSE geometry is drawn last.
          renderSortKey: isHighlighted ? 1000 : 0,
          facilityIcon,
        },
      };
    },

    renderMapData() {
      if (!this.$map || !this.$map.getSource(PROJECT_SOURCE_ID)) return;

      const rendered = this.allProjectFeatures
        .map((feature, sourceIndex) => ({
          feature: this.getStyledFeature(feature),
          sourceIndex,
        }))
        .sort((a, b) => {
          const aSortKey = Number(a.feature.properties?.renderSortKey ?? 0);
          const bSortKey = Number(b.feature.properties?.renderSortKey ?? 0);

          // Keep normal features in their original order, but move the
          // highlighted project to the end of the GeoJSON source. This is a
          // fallback for any renderer that ignores line-sort-key.
          return aSortKey - bSortKey || a.sourceIndex - b.sourceIndex;
        })
        .map(({ feature }) => feature);

      this.renderedProjectFeatures = rendered;

      const source = this.$map.getSource(PROJECT_SOURCE_ID);
      if (source && source.setData) {
        source.setData({
          type: "FeatureCollection",
          features: rendered,
        });
      }

      if (this.isConceptMapMode()) {
        this.syncConceptMapLayers();
      }
    },

    onProjectMouseEnter(event) {
      if (this.isConceptMapMode()) {
        if (this.$map) this.$map.getCanvas().style.cursor = "";
        if (this.mapPopup) this.mapPopup.remove();
        return;
      }
      if (!event.features || !event.features.length) return;
      const feature = event.features[0];
      const isFacilitiesPoint =
        this.getProperty(feature, "tse_geometry_role") === "facilities" &&
        feature.geometry &&
        feature.geometry.type === "Point";

      if (isFacilitiesPoint) {
        if (this.mapPopup) this.mapPopup.remove();
        return;
      }

      this.$map.getCanvas().style.cursor = "pointer";
      this.hoveredFeatureKey = this.getProperty(feature, "_key");
      this.renderMapData();

      if (
        this.getProperty(feature, "typology_en") !== "Pocket Parks" &&
        this.getProperty(feature, "typology_en") !== "Neighborhood Parks"
      ) {
        this.setShapesInfoWindow(feature, event.lngLat);
      }
    },

    onProjectMouseMove(event) {
      if (this.isConceptMapMode()) return;
      if (!event.features || !event.features.length) return;
      const feature = event.features[0];
      if (
        this.mapPopup &&
        this.mapPopup.isOpen &&
        this.mapPopup.isOpen() &&
        event.lngLat
      ) {
        this.mapPopup.setLngLat(event.lngLat);
      } else {
        this.setShapesInfoWindow(feature, event.lngLat);
      }
    },

    onProjectMouseLeave() {
      if (this.$map) this.$map.getCanvas().style.cursor = "";
      this.hoveredFeatureKey = null;
      this.renderMapData();
      if (this.mapPopup) this.mapPopup.remove();
    },

    onProjectClick(event) {
      if (this.isConceptMapMode()) return;
      if (!event.features || !event.features.length) return;
      this.routeToProjectDetails(event.features[0]);
    },

    whenHoverOnMap() {
      // Layer mouse events are registered in bindProjectLayerEvents().
    },

    whenClickOnMap() {
      // Layer click events are registered in bindProjectLayerEvents().
    },

    routeToProjectDetails(feature) {
      const typology_en = this.getProperty(feature, "typology_en");

      if (
        typology_en != "Pocket Parks" &&
        typology_en != "Neighborhood Parks"
      ) {
        const id =
          this.getProperty(feature, "parent_id") ||
          this.getProperty(feature, "id");

        let params = {
          id: id,
          back_router: String(typology_en || "").replaceAll(" ", "_"),
        };

        if (this.$route.name == "Home") {
          params = {
            id: id,
            back_router: String(typology_en || "").replaceAll(" ", "_"),
            back_to: "home",
          };
        }

        this.$router.push({
          name: "projects",
          params: params,
        });
      }
    },

    syncMapPopupTheme() {
      if (!this.mapPopup || typeof this.mapPopup.getElement !== "function") {
        return;
      }

      const popupElement = this.mapPopup.getElement();
      if (!popupElement) return;

      const isDark = this.theme === "dark";
      popupElement.classList.toggle("is-dark-mode", isDark);

      // Mapbox creates popup DOM outside this Vue component's scoped CSS.
      // Apply the theme directly so dark mode cannot be blocked by scoped
      // selectors or Mapbox's default white popup background.
      const content = popupElement.querySelector(".mapboxgl-popup-content");
      if (content) {
        content.style.setProperty(
          "background-color",
          isDark ? "#2c2c2c" : "#ffffff",
          "important",
        );
        content.style.setProperty(
          "color",
          isDark ? "#ffffff" : "#111111",
          "important",
        );
        content.style.setProperty(
          "border",
          isDark ? "1px solid #555555" : "1px solid transparent",
          "important",
        );
        content.style.setProperty(
          "box-shadow",
          isDark
            ? "0 8px 22px rgba(0, 0, 0, 0.45)"
            : "0 8px 22px rgba(0, 0, 0, 0.2)",
          "important",
        );
      }

      popupElement
        .querySelectorAll(".marker-tool, .marker-tool *")
        .forEach((el) => {
          el.style.setProperty(
            "color",
            isDark ? "#ffffff" : "#111111",
            "important",
          );
        });

      // Project title uses the brighter Green Riyadh green in dark mode and
      // the darker official-logo green in light mode for better contrast.
      popupElement
        .querySelectorAll(
          ".map-project-tooltip__title, .map-project-tooltip__title *",
        )
        .forEach((el) => {
          el.style.setProperty(
            "color",
            isDark ? "#86dc4f" : "#17743a",
            "important",
          );
        });

      const tip = popupElement.querySelector(".mapboxgl-popup-tip");
      if (tip) {
        const tipColor = isDark ? "#2c2c2c" : "#ffffff";
        [
          "border-top-color",
          "border-bottom-color",
          "border-left-color",
          "border-right-color",
        ].forEach((property) => tip.style.removeProperty(property));

        if (popupElement.classList.contains("mapboxgl-popup-anchor-top")) {
          tip.style.setProperty("border-bottom-color", tipColor, "important");
        } else if (
          popupElement.classList.contains("mapboxgl-popup-anchor-bottom")
        ) {
          tip.style.setProperty("border-top-color", tipColor, "important");
        } else if (
          popupElement.classList.contains("mapboxgl-popup-anchor-left")
        ) {
          tip.style.setProperty("border-right-color", tipColor, "important");
        } else if (
          popupElement.classList.contains("mapboxgl-popup-anchor-right")
        ) {
          tip.style.setProperty("border-left-color", tipColor, "important");
        }
      }
    },

    setShapesInfoWindow(feature, lngLat) {
      if (!this.mapPopup || !lngLat) return;

      const name_en = this.getProperty(feature, "name_en") || "";
      const trees_planted = this.getProperty(feature, "trees_planted");
      const trees_target = this.getProperty(feature, "trees_target");
      const progress_actual = this.getProperty(feature, "progress_actual");
      const progress_planned = this.getProperty(feature, "progress_planned");
      const contract_end_date = this.getFeatureContractEndDisplay(feature);
      const isTse =
        this.isTseTypology(this.getProperty(feature, "typology_en")) ||
        this.getProperty(feature, "tse_geometry_role") != null;

      const tpNum = Number(String(trees_planted ?? "").replace(/[^0-9.]/g, ""));
      const ttNum = Number(String(trees_target ?? "").replace(/[^0-9.]/g, ""));
      const treesActual = Number.isFinite(tpNum) ? tpNum : 0;
      const treesPlanned = Number.isFinite(ttNum) ? ttNum : 0;
      const treesValue = this.formatTooltipTrees({
        trees_planted: treesActual,
        trees_target: treesPlanned,
      });

      const hasConstruction =
        progress_actual !== undefined &&
        progress_actual !== null &&
        progress_actual !== "" &&
        !isNaN(Number(progress_actual)) &&
        Number(progress_actual) < 100;

      const paNum = Number(
        String(progress_actual ?? "").replace(/[^0-9.]/g, ""),
      );
      const ppNum = Number(
        String(progress_planned ?? "").replace(/[^0-9.]/g, ""),
      );
      const progressActualDisplay = Number.isFinite(paNum)
        ? `${Math.trunc(paNum)}%`
        : `${progress_actual}%`;
      const progressPlannedDisplay =
        Number.isFinite(ppNum) && ppNum >= 0 ? `${ppNum}%` : null;
      const progressText = progressPlannedDisplay
        ? `${progressActualDisplay}/${progressPlannedDisplay}`
        : progressActualDisplay;

      const treesLine = isTse
        ? ""
        : `<div style="margin: 0 0 6px 0;"><b>Trees Planted:</b> ${treesValue}</div>`;

      const infoWindowId = `map-tooltip-${String(
        this.getProperty(feature, "parent_id") ||
          this.getProperty(feature, "id") ||
          "project",
      ).replace(/[^a-zA-Z0-9_-]/g, "")}`;

      const contentString = `
        <div
          id="${infoWindowId}"
          class="marker-tool"
          style="padding-bottom: 8px; line-height: 1.25; cursor: pointer; color: ${this.theme === "dark" ? "#ffffff" : "#111111"};"
          role="button"
          tabindex="0"
        >
          <div id="bodyContent" style="padding-bottom: 6px; color: ${this.theme === "dark" ? "#ffffff" : "#111111"};">
            <h4 class="map-project-tooltip__title" style="margin: 0 0 10px 0; color: ${this.theme === "dark" ? "#86dc4f" : "#17743a"};"><b>${name_en}</b></h4>
            <div style="padding-left: 10px; padding-right: 6px; color: ${this.theme === "dark" ? "#ffffff" : "#111111"};">
              ${treesLine}
              ${
                hasConstruction
                  ? `
                <div style="margin-top:6px; color: ${this.theme === "dark" ? "#ffffff" : "#111111"};">
                  <div style="margin: 0 0 6px 0;"><b>Progress:</b> ${progressText}</div>
                  ${contract_end_date ? `<div style=\"margin: 0 0 2px 0;\"><b>Contract End:</b> ${contract_end_date}</div>` : ""}
                </div>`
                  : ""
              }
            </div>
          </div>
        </div>
      `;

      this.mapPopup.setLngLat(lngLat).setHTML(contentString).addTo(this.$map);
      this.syncMapPopupTheme();

      window.setTimeout(() => {
        const tooltipEl = document.getElementById(infoWindowId);
        if (!tooltipEl) return;

        tooltipEl.addEventListener("click", () => {
          this.routeToProjectDetails(feature);
        });

        tooltipEl.addEventListener("keydown", (e) => {
          if (e.key === "Enter" || e.key === " ") {
            e.preventDefault();
            this.routeToProjectDetails(feature);
          }
        });
      }, 0);
    },

    initMarkerIcons() {
      this.icons.active.icon = "/images/marker-selected.gif";
      this.icons.inactive.icon =
        "/images/marker-unselected-" + this.theme + "2.png";
    },

    processPoints(geometry, callback, thisArg) {
      if (!geometry) return;
      this.eachCoordinate(geometry.coordinates || geometry, (coord) => {
        callback.call(thisArg, { lng: coord[0], lat: coord[1] });
      });
    },

    eachCoordinate(coords, callback) {
      if (!Array.isArray(coords)) return;
      if (
        coords.length >= 2 &&
        typeof coords[0] === "number" &&
        typeof coords[1] === "number"
      ) {
        callback(coords);
        return;
      }
      coords.forEach((item) => this.eachCoordinate(item, callback));
    },

    fitMapToProjectIds(projectIds, maxZoom = 13, options = {}) {
      const ids = (Array.isArray(projectIds) ? projectIds : [projectIds])
        .map((id) => this.normalizeProjectId(id))
        .filter(Boolean);
      if (!ids.length) return;

      const features = this.allProjectFeatures.filter((feature) =>
        ids.includes(
          this.normalizeProjectId(
            feature.properties.parent_id ?? feature.properties.id,
          ),
        ),
      );
      this.fitFeatures(features, maxZoom, options);
    },

    fitFeatures(features, maxZoom = 13, options = {}) {
      if (this.isBaseMapStyleSwitching) {
        this.restoreMapCameraStateImmediately();
        return;
      }

      if (this.isStandaloneThreeMode) {
        this.resetThreeParkCamera();
        return;
      }

      if (
        !this.$map ||
        !this.mapboxgl ||
        !Array.isArray(features) ||
        !features.length
      )
        return;

      const bounds = new this.mapboxgl.LngLatBounds();
      let hasBounds = false;

      features.forEach((feature) => {
        if (!feature || !feature.geometry) return;
        this.eachCoordinate(feature.geometry.coordinates, (coord) => {
          if (
            Array.isArray(coord) &&
            coord.length >= 2 &&
            Number.isFinite(Number(coord[0])) &&
            Number.isFinite(Number(coord[1]))
          ) {
            bounds.extend([Number(coord[0]), Number(coord[1])]);
            hasBounds = true;
          }
        });
      });

      if (!hasBounds) return;

      const forceStreetCamera = Boolean(options && options.forceStreetCamera);
      const isThreeDimensionalMode =
        this.isThreeDimensionalMapMode() && !forceStreetCamera;
      const targetPitch = isThreeDimensionalMode ? MAPBOX_3D_TERRAIN_PITCH : 0;
      const targetBearing = isThreeDimensionalMode
        ? MAPBOX_3D_TERRAIN_BEARING
        : 0;
      let cameraAngleRestored = false;
      const restoreCameraAngle = () => {
        if (cameraAngleRestored || !this.$map) return;
        cameraAngleRestored = true;
        this.$map.easeTo({
          pitch: targetPitch,
          bearing: targetBearing,
          duration: 0,
        });
      };

      try {
        this.$map.once("moveend", restoreCameraAngle);
      } catch (error) {
        // If this Mapbox instance cannot register the one-time event, the
        // timeout fallback below still restores the camera angle.
      }

      this.$map.fitBounds(bounds, {
        padding: 40,
        duration: 350,
        maxZoom,
        pitch: targetPitch,
        bearing: targetBearing,
      });

      window.setTimeout(restoreCameraAngle, 390);
    },
  },
};
</script>

<style>
.map-component-wrap {
  position: relative;
}

.map-component-wrap.is-map-expanded {
  position: fixed;
  top: 0;
  right: 0;
  bottom: 0;
  left: var(--green-riyadh-expanded-map-left, 0px);
  z-index: 2147483000;
  width: auto;
  height: 100vh;
  overflow: hidden;
  background: #061218;
}

.map-component-wrap.is-map-expanded #map {
  top: 0 !important;
  right: 0 !important;
  bottom: 0 !important;
  left: var(--green-riyadh-expanded-map-left, 0px) !important;
  width: auto !important;
  height: 100vh !important;
  z-index: 2147483001 !important;
}

.map-component-wrap.is-map-expanded .three-park-host {
  z-index: 2147483002;
}

.map-component-wrap.is-map-expanded .main-map-style-switch,
.map-component-wrap.is-map-expanded .three-tree-type-panel,
.map-component-wrap.is-map-expanded .three-layer-filters,
.map-component-wrap.is-map-expanded .main-map-layer-switch,
.map-component-wrap.is-map-expanded .tse-map-legend,
.map-component-wrap.is-map-expanded .map-location-message,
.map-component-wrap.is-map-expanded .map-navigation-controls {
  z-index: 2147483646 !important;
}

html.green-riyadh-map-expanded,
body.green-riyadh-map-expanded {
  width: 100% !important;
  height: 100% !important;
  overflow: hidden !important;
  overscroll-behavior: none;
}

body.green-riyadh-map-expanded {
  scrollbar-width: none;
}

body.green-riyadh-map-expanded::-webkit-scrollbar {
  display: none;
  width: 0;
  height: 0;
}

.three-park-host {
  position: fixed;
  z-index: 2;
  overflow: hidden;
  background: #061218;
  touch-action: none;
}

.three-park-canvas {
  position: absolute;
  inset: 0;
  display: block;
  width: 100%;
  height: 100%;
  outline: none;
}

.three-park-canvas.is-three-elevation-dragging {
  cursor: ns-resize !important;
}

.three-park-status {
  position: absolute;
  z-index: 3;
  top: 50%;
  left: 50%;
  max-width: min(320px, calc(100% - 32px));
  transform: translate(-50%, -50%);
  padding: 12px 16px;
  border-radius: 10px;
  background: rgba(4, 20, 27, 0.9);
  border: 1px solid rgba(92, 236, 255, 0.42);
  color: #d8fbff;
  font-size: 13px;
  font-weight: 700;
  line-height: 1.4;
  text-align: center;
  box-sizing: border-box;
}

.three-park-status--error {
  border-color: rgba(255, 159, 122, 0.6);
  color: #ffe1d7;
}

.three-park-attribution {
  position: absolute;
  z-index: 3;
  left: 8px;
  bottom: 6px;
  padding: 2px 5px;
  border-radius: 3px;
  background: rgba(1, 7, 11, 0.66);
  color: rgba(224, 244, 247, 0.82);
  font-size: 10px;
  line-height: 1.2;
  pointer-events: none;
}

.three-park-compass {
  position: absolute;
  z-index: 5;
  left: 20px;
  bottom: 34px;
  width: 94px;
  height: 94px;
  pointer-events: none;
  user-select: none;
  filter: drop-shadow(0 5px 12px rgba(0, 0, 0, 0.42));
}

.three-park-compass__dial {
  position: relative;
  width: 100%;
  height: 100%;
  border: 1px solid rgba(70, 220, 239, 0.42);
  border-radius: 50%;
  background: radial-gradient(
    circle at center,
    rgba(4, 20, 27, 0.94) 0 42%,
    rgba(12, 65, 76, 0.58) 43% 45%,
    rgba(2, 13, 18, 0.92) 46% 61%,
    rgba(49, 197, 218, 0.48) 62% 64%,
    rgba(2, 12, 17, 0.82) 65% 100%
  );
  box-shadow:
    inset 0 0 0 4px rgba(5, 35, 43, 0.78),
    inset 0 0 18px rgba(44, 211, 235, 0.14),
    0 0 0 1px rgba(4, 14, 19, 0.9);
}

.three-park-compass__dial::before,
.three-park-compass__dial::after {
  content: "";
  position: absolute;
  border-radius: 50%;
  pointer-events: none;
}

.three-park-compass__dial::before {
  inset: 7px;
  border: 1px solid rgba(63, 213, 235, 0.36);
  background: repeating-conic-gradient(
    from -0.5deg,
    rgba(90, 231, 247, 0.72) 0deg 1deg,
    transparent 1deg 29deg,
    rgba(34, 137, 154, 0.56) 29deg 30deg
  );
  -webkit-mask: radial-gradient(circle, transparent 0 82%, #000 83% 100%);
  mask: radial-gradient(circle, transparent 0 82%, #000 83% 100%);
}

.three-park-compass__dial::after {
  inset: 15px;
  border: 1px solid rgba(60, 204, 224, 0.25);
  box-shadow: 0 0 8px rgba(41, 216, 239, 0.12);
}

.three-park-compass__rose {
  position: absolute;
  inset: 11px;
  transform: rotate(0deg);
  transform-origin: 50% 50%;
  transition: transform 70ms linear;
  will-change: transform;
}

.three-park-compass__north {
  position: absolute;
  top: 4px;
  left: 50%;
  transform: translateX(-50%);
  color: rgba(111, 236, 250, 0.88);
  font-size: 12px;
  font-weight: 800;
  line-height: 1;
  text-shadow: 0 0 7px rgba(49, 224, 246, 0.55);
}

.three-park-compass__arrow {
  position: absolute;
  top: 19px;
  left: 50%;
  width: 34px;
  height: 43px;
  transform: translateX(-50%);
  overflow: visible;
  filter: drop-shadow(0 0 5px rgba(36, 225, 249, 0.42));
}

.three-park-compass__arrow-main {
  fill: #25d8f2;
  stroke: rgba(122, 246, 255, 0.86);
  stroke-width: 0.8;
  stroke-linejoin: round;
}

.three-park-compass__arrow-cut {
  fill: rgba(2, 28, 36, 0.88);
}

.three-park-compass__center-dot {
  position: absolute;
  left: 50%;
  top: 50%;
  width: 4px;
  height: 4px;
  transform: translate(-50%, -50%);
  border-radius: 50%;
  background: #80f5ff;
  box-shadow: 0 0 7px rgba(65, 229, 249, 0.9);
}

.three-tree-type-panel {
  position: fixed;
  z-index: 10000;
  top: calc(var(--green-riyadh-map-control-top, 18px) + 52px);
  left: var(--green-riyadh-map-control-left, 14px);
  width: min(
    320px,
    calc(100vw - var(--green-riyadh-map-control-left, 14px) - 28px)
  );
  max-height: min(
    440px,
    var(--green-riyadh-map-tree-panel-max-height, calc(100vh - 96px))
  );
  overflow: hidden;
  border: 1px solid rgba(67, 211, 230, 0.3);
  border-radius: 12px;
  background: rgba(3, 17, 23, 0.9);
  box-shadow: 0 14px 30px rgba(0, 0, 0, 0.34);
  color: #dffcff;
  backdrop-filter: blur(9px);
  pointer-events: auto;
}

.three-tree-type-panel__header {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 10px;
  padding: 12px 12px 9px;
  border-bottom: 1px solid rgba(91, 219, 235, 0.18);
}

.three-tree-type-panel__title {
  margin: 0;
  color: #f1feff;
  font-size: 13px;
  font-weight: 900;
  line-height: 1.2;
  text-transform: uppercase;
}

.three-tree-type-panel__total {
  margin: 4px 0 0;
  color: rgba(195, 241, 247, 0.78);
  font-size: 10px;
  font-weight: 700;
}

.three-tree-type-panel__total strong {
  color: #75eef9;
  font-size: 11px;
}

.three-tree-type-panel__close {
  width: 25px;
  height: 25px;
  padding: 0;
  border: 1px solid rgba(92, 226, 242, 0.2);
  border-radius: 7px;
  background: rgba(6, 35, 43, 0.58);
  color: rgba(204, 249, 253, 0.86);
  cursor: pointer;
  font-size: 18px;
  line-height: 21px;
}

.three-tree-type-panel__close:hover,
.three-tree-type-panel__close:focus-visible {
  outline: none;
  border-color: rgba(111, 241, 255, 0.68);
  background: rgba(10, 64, 76, 0.72);
}

.three-tree-type-panel__message {
  padding: 15px 12px;
  color: rgba(196, 237, 242, 0.78);
  font-size: 10px;
  font-weight: 700;
}

.three-tree-type-panel__list {
  max-height: 365px;
  overflow-y: auto;
  padding: 5px 6px 8px;
  scrollbar-width: thin;
}

.three-tree-type-panel__row {
  display: grid;
  grid-template-columns: 38px minmax(0, 1fr) auto;
  align-items: center;
  gap: 8px;
  min-height: 43px;
  padding: 4px 6px;
  border-bottom: 1px solid rgba(84, 206, 222, 0.1);
}

.three-tree-type-panel__row:last-child {
  border-bottom: 0;
}

.three-tree-type-panel__icon {
  position: relative;
  display: inline-block;
  width: 34px;
  height: 34px;
}

.three-tree-type-panel__canopy {
  position: absolute;
  z-index: 2;
  left: 4px;
  top: 2px;
  width: 26px;
  height: 21px;
  border: 1px solid rgba(97, 231, 241, 0.58);
  background: var(--tree-icon-color, #0f412c);
  clip-path: polygon(
    50% 0,
    81% 9%,
    100% 38%,
    94% 72%,
    70% 95%,
    31% 100%,
    6% 76%,
    0 42%,
    18% 13%
  );
  box-shadow: inset 0 0 7px rgba(95, 231, 241, 0.12);
}

.three-tree-type-panel__trunk {
  position: absolute;
  z-index: 1;
  left: 15px;
  top: 20px;
  width: 5px;
  height: 13px;
  border: 1px solid rgba(70, 210, 220, 0.5);
  background: #0b4a3b;
  transform-origin: top center;
}

.three-tree-type-panel__icon.is-umbrella .three-tree-type-panel__canopy {
  left: 1px;
  top: 5px;
  width: 32px;
  height: 16px;
  clip-path: polygon(
    4% 63%,
    14% 29%,
    37% 8%,
    68% 4%,
    91% 27%,
    100% 63%,
    74% 83%,
    26% 87%
  );
}

.three-tree-type-panel__icon.is-columnar .three-tree-type-panel__canopy {
  left: 8px;
  top: 0;
  width: 18px;
  height: 25px;
  clip-path: polygon(
    50% 0,
    80% 12%,
    100% 45%,
    84% 82%,
    50% 100%,
    16% 82%,
    0 45%,
    20% 12%
  );
}

.three-tree-type-panel__icon.is-open .three-tree-type-panel__canopy {
  opacity: 0.9;
}

.three-tree-type-panel__name {
  min-width: 0;
  overflow: hidden;
  color: #d9f8fa;
  font-size: 10px;
  font-weight: 700;
  line-height: 1.25;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.three-tree-type-panel__count {
  min-width: 28px;
  padding: 3px 6px;
  border-radius: 999px;
  background: rgba(20, 108, 119, 0.42);
  color: #8ef5fc;
  font-size: 10px;
  font-weight: 900;
  text-align: center;
}

.three-tree-type-panel.is-arabic .three-tree-type-panel__row {
  grid-template-columns: 38px minmax(0, 1fr) auto;
}

.three-park-tool-panel {
  position: absolute;
  z-index: 6;
  left: 24px;
  top: 50%;
  display: flex;
  flex-direction: column;
  gap: 10px;
  padding: 10px 9px;
  transform: translateY(-50%);
  border: 1px solid rgba(71, 221, 240, 0.28);
  border-radius: 10px;
  background: rgba(2, 12, 17, 0.7);
  box-shadow: 0 12px 22px rgba(0, 0, 0, 0.28);
  backdrop-filter: blur(7px);
}

.three-park-tool-panel__button {
  width: 34px;
  height: 34px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  border: 1px solid rgba(88, 230, 247, 0.18);
  border-radius: 8px;
  background: rgba(2, 12, 17, 0.28);
  color: rgba(127, 240, 251, 0.9);
  cursor: pointer;
  transition:
    border-color 0.18s ease,
    background-color 0.18s ease,
    box-shadow 0.18s ease,
    transform 0.18s ease;
}

.three-park-tool-panel__button:hover:not(.is-disabled),
.three-park-tool-panel__button:focus-visible:not(.is-disabled) {
  outline: none;
  border-color: rgba(109, 241, 255, 0.68);
  background: rgba(10, 54, 66, 0.6);
  box-shadow: 0 0 0 1px rgba(22, 201, 223, 0.18);
}

.three-park-tool-panel__button.is-active {
  border-color: rgba(104, 240, 255, 0.82);
  background: rgba(11, 78, 95, 0.72);
  box-shadow:
    inset 0 0 12px rgba(47, 220, 240, 0.18),
    0 0 0 1px rgba(18, 171, 192, 0.2);
}

.three-park-tool-panel__button.is-disabled {
  opacity: 0.42;
  cursor: default;
}

.three-park-tool-panel__button svg {
  width: 19px;
  height: 19px;
  fill: none;
  stroke: currentColor;
  stroke-width: 1.7;
  stroke-linecap: round;
  stroke-linejoin: round;
}

.three-park-label-overlay {
  position: absolute;
  inset: 0;
  z-index: 7;
  pointer-events: none;
}

.three-park-label-overlay__item {
  position: absolute;
  max-width: 220px;
  padding: 5px 8px 4px;
  border: 1px solid rgba(104, 238, 255, 0.36);
  border-radius: 999px;
  background: rgba(2, 16, 23, 0.84);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
  color: #d7fbff;
  font-size: 10px;
  font-weight: 700;
  line-height: 1.2;
  white-space: nowrap;
  overflow: visible;
  /* Structure APP_LABEL text is authored in ArcGIS. Never let a parent/global
     style convert its capitalization. */
  text-transform: none !important;
  transform: translate(-50%, calc(-100% - 10px));
  will-change: left, top;
}

.three-park-label-overlay__item::after {
  content: "";
  position: absolute;
  left: 50%;
  bottom: -7px;
  width: 1px;
  height: 7px;
  transform: translateX(-50%);
  background: linear-gradient(
    to bottom,
    rgba(104, 238, 255, 0.88),
    rgba(104, 238, 255, 0)
  );
}

#map.is-hidden-for-three {
  visibility: hidden;
  pointer-events: none;
}

/* Responsive safety: fixed map controls must never float over the page when
   the responsive layout leaves too little of the actual map visible. */
.map-component-wrap .is-outside-visible-map {
  visibility: hidden !important;
  pointer-events: none !important;
}

/* Pull-curtain control for the Layer Filters panel. It is intentionally
   styled as a soft folded curtain/valance rather than a conventional button. */
.three-layer-filters-curtain {
  position: fixed;
  top: calc(var(--green-riyadh-map-filter-top, 18px) - 7px);
  right: var(--green-riyadh-map-filter-right, 14px);
  z-index: 2147483647;
  width: min(244px, var(--green-riyadh-map-layer-filter-width, 244px));
  height: 17px;
  margin: 0;
  padding: 0 18px;
  border: 0;
  border-radius: 8px 8px 5px 5px;
  outline: none;
  background: transparent;
  box-shadow: none;
  color: var(--qad-ui-green-bright, #99ea61);
  cursor: ns-resize;
  pointer-events: auto;
  appearance: none;
  -webkit-appearance: none;
}

.three-layer-filters-curtain__folds {
  position: absolute;
  inset: 2px 0 1px;
  border: 1px solid rgba(134, 220, 79, 0.34);
  border-radius: 7px 7px 4px 4px;
  background:
    linear-gradient(to bottom, rgba(255,255,255,0.10), rgba(0,0,0,0.28)),
    repeating-linear-gradient(
      90deg,
      rgba(21, 27, 24, 0.96) 0px,
      rgba(48, 54, 50, 0.96) 7px,
      rgba(15, 20, 18, 0.98) 14px
    );
  box-shadow:
    inset 0 1px 1px rgba(255,255,255,0.08),
    0 2px 5px rgba(0,0,0,0.34);
  transition: transform 220ms ease, filter 220ms ease;
}

.three-layer-filters-curtain__pull {
  position: absolute;
  left: 50%;
  top: 50%;
  z-index: 1;
  transform: translate(-50%, -46%);
  font-size: 10px;
  line-height: 1;
  color: var(--qad-ui-green-bright, #99ea61);
  text-shadow: 0 0 7px rgba(134,220,79,0.55);
  transition: transform 220ms ease;
}

.three-layer-filters-curtain:hover .three-layer-filters-curtain__folds,
.three-layer-filters-curtain:focus-visible .three-layer-filters-curtain__folds {
  filter: brightness(1.14);
}

.three-layer-filters-curtain:focus-visible .three-layer-filters-curtain__folds {
  box-shadow:
    inset 0 1px 1px rgba(255,255,255,0.08),
    0 0 0 1px var(--qad-ui-green, #86dc4f),
    0 2px 7px rgba(0,0,0,0.34);
}

.three-layer-filters-curtain.is-curtain-up .three-layer-filters-curtain__folds {
  transform: scaleY(0.72);
}

.three-layer-filters {
  position: fixed;
  top: var(--green-riyadh-map-filter-top, 18px);
  right: var(--green-riyadh-map-filter-right, 14px);
  z-index: 2147483646;
  width: min(244px, var(--green-riyadh-map-layer-filter-width, 244px));
  max-height: var(--green-riyadh-map-overlay-max-height, calc(100vh - 36px));
  overflow-y: auto;
  overscroll-behavior: contain;
  scrollbar-width: thin;
  padding: 12px 12px 10px;
  border: 1px solid rgba(93, 235, 255, 0.28);
  border-radius: 10px;
  background: rgba(2, 12, 17, 0.9);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.32);
  color: #d9f9fc;
  pointer-events: auto;
  box-sizing: border-box;
  backdrop-filter: blur(8px);
  text-transform: uppercase;
  transform-origin: top center;
  transition:
    transform 280ms cubic-bezier(0.2, 0.8, 0.2, 1),
    opacity 180ms ease;
}

.three-layer-filters.is-curtain-hidden {
  transform: translateY(calc(-100% - 30px)) scaleY(0.96);
  opacity: 0;
  pointer-events: none;
}

.three-layer-filters.is-arabic {
  text-align: right;
}

.three-layer-filters__header {
  display: grid;
  grid-template-columns: minmax(0, 1fr) 13px 13px;
  align-items: center;
  column-gap: 8px;
  margin: 0 0 9px;
}

.three-layer-filters__title {
  grid-column: 1;
  margin: 0;
  color: #78eefe;
  font-size: 11px;
  font-weight: 800;
  letter-spacing: 0.04em;
  line-height: 1.2;
  text-transform: uppercase;
}

.three-layer-filters__visibility-column-title,
.three-layer-filters__label-column-title {
  display: inline-flex;
  width: 13px;
  height: 13px;
  align-items: center;
  justify-content: center;
  color: #78eefe;
}

.three-layer-filters__visibility-column-title {
  grid-column: 2;
}

.three-layer-filters__label-column-title {
  grid-column: 3;
}

.three-layer-filters__visibility-column-title svg,
.three-layer-filters__label-column-title svg {
  width: 13px;
  height: 13px;
  overflow: visible;
  fill: none;
  stroke: currentColor;
  stroke-width: 1.8;
  stroke-linecap: round;
  stroke-linejoin: round;
}

.three-layer-filters__label-column-title circle {
  fill: currentColor;
  stroke: none;
}

.three-layer-filters__visibility-column-title circle {
  fill: none;
}

.three-layer-filter-block + .three-layer-filter-block {
  margin-top: 1px;
}

.three-layer-filter-row {
  display: flex;
  align-items: center;
  min-height: 26px;
  gap: 5px;
  color: #d9f9fc;
  font-size: 11px;
  font-weight: 700;
  line-height: 1.2;
  user-select: none;
}

.three-layer-filter-row__content {
  display: flex;
  align-items: center;
  justify-content: space-between;
  min-width: 0;
  flex: 1 1 auto;
  gap: 8px;
  cursor: pointer;
}

.three-layer-filter-row__label-toggle {
  display: inline-flex;
  width: 13px;
  height: 13px;
  flex: 0 0 13px;
  align-items: center;
  justify-content: center;
  cursor: pointer;
}

.three-layer-filter-row__label-column-placeholder {
  display: inline-block;
  width: 13px;
  height: 13px;
  flex: 0 0 13px;
}

.three-layer-filter-row__label-checkbox {
  accent-color: #78eefe;
}

.three-layer-filter-row__arrow,
.three-layer-filter-row__arrow-placeholder {
  width: 15px;
  height: 20px;
  flex: 0 0 15px;
}

.three-layer-filter-row__arrow {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  border: 0;
  background: transparent;
  color: #78eefe;
  cursor: pointer;
  font: inherit;
}

.three-layer-filter-row__arrow span {
  display: block;
  font-size: 17px;
  line-height: 1;
  transform: rotate(0deg);
  transition: transform 0.16s ease;
}

.three-layer-filter-row__arrow.is-expanded span {
  transform: rotate(90deg);
}

.three-layer-filter-row__arrow:disabled {
  color: rgba(133, 154, 158, 0.35);
  cursor: not-allowed;
}

.three-layer-filter-row__label {
  display: inline-flex;
  align-items: center;
  min-width: 0;
  gap: 7px;
}

.three-layer-filter-row__icon {
  width: 8px;
  height: 8px;
  flex: 0 0 8px;
  border: 1px solid rgba(79, 226, 244, 0.72);
  border-radius: 2px;
  box-shadow: inset 0 0 0 1px rgba(10, 49, 57, 0.9);
}

.three-layer-filter-row__checkbox {
  width: 13px;
  height: 13px;
  flex: 0 0 13px;
  margin: 0;
  accent-color: #39e9ff;
  cursor: pointer;
}

.three-layer-filter-row.is-disabled {
  color: rgba(190, 208, 211, 0.48);
  cursor: not-allowed;
}

.three-layer-filter-row.is-disabled .three-layer-filter-row__icon {
  border-color: rgba(133, 154, 158, 0.35);
  box-shadow: none;
}

.three-layer-filter-row.is-disabled .three-layer-filter-row__checkbox {
  cursor: not-allowed;
  opacity: 0.48;
}

.three-layer-filter-row.is-locked {
  color: #d9f9fc;
  cursor: pointer;
}

.three-layer-filter-row.is-locked .three-layer-filter-row__icon {
  border-color: rgba(79, 226, 244, 0.72);
  box-shadow: inset 0 0 0 1px rgba(10, 49, 57, 0.9);
}

.three-layer-filter-row.is-locked .three-layer-filter-row__checkbox {
  cursor: pointer;
  opacity: 1;
}

.three-layer-filter-row.is-loading {
  opacity: 0.72;
}

.three-layer-filter-row.is-partial .three-layer-filter-row__icon {
  background: rgba(57, 233, 255, 0.34);
}

.three-layer-filter-children {
  margin: 1px 0 4px 20px;
  padding: 3px 0 3px 9px;
  border-left: 1px solid rgba(79, 226, 244, 0.24);
}

.three-layer-filters.is-arabic .three-layer-filter-children {
  margin: 1px 20px 4px 0;
  padding: 3px 9px 3px 0;
  border-left: 0;
  border-right: 1px solid rgba(79, 226, 244, 0.24);
}

.three-layer-filter-child-row {
  display: grid;
  grid-template-columns: minmax(0, 1fr) 13px 13px;
  align-items: center;
  min-height: 23px;
  column-gap: 8px;
  color: rgba(217, 249, 252, 0.88);
  font-size: 9px;
  font-weight: 650;
  line-height: 1.2;
}

.three-layer-filter-child-row__label {
  min-width: 0;
  overflow-wrap: anywhere;
  /* Main filter names stay uppercase; expandable sub-layer labels keep
     their authored Title Case, such as "Pedestrian Space" and "Sidewalk". */
  text-transform: none;
}

.three-layer-filters__error {
  margin: 7px 0 0;
  color: #ffd2c5;
  font-size: 9px;
  font-weight: 700;
  line-height: 1.35;
}

.main-map-style-switch {
  position: fixed;
  top: var(--green-riyadh-map-control-top, 18px);
  left: var(--green-riyadh-map-control-left, 14px);
  z-index: 10001;
  display: flex;
  max-width: var(--green-riyadh-map-style-max-width, calc(100vw - 28px));
  height: 42px;
  overflow: hidden;
  border-radius: 10px;
  background: rgba(255, 255, 255, 0.94);
  border: 1px solid rgba(0, 0, 0, 0.08);
  box-shadow: 0 6px 16px rgba(0, 0, 0, 0.12);
  pointer-events: auto;
  box-sizing: border-box;
}

.qadisiyah-concept-switch {
  position: fixed;
  top: calc(var(--green-riyadh-map-control-top, 18px) + 50px);
  left: var(--green-riyadh-map-control-left, 14px);
  z-index: 10001;
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 6px;
  max-width: min(
    520px,
    calc(100vw - var(--green-riyadh-map-control-left, 14px) - 24px)
  );
  min-height: 38px;
  padding: 7px 8px;
  overflow: hidden;
  border-radius: 10px;
  background: rgba(255, 255, 255, 0.92);
  border: 1px solid rgba(0, 0, 0, 0.08);
  box-shadow: 0 6px 16px rgba(0, 0, 0, 0.12);
  pointer-events: auto;
  box-sizing: border-box;
  backdrop-filter: blur(6px);
}

.qadisiyah-concept-switch__label {
  color: #2f3a2f;
  font-size: 11px;
  font-weight: 900;
  line-height: 1;
  margin: 0 2px;
  white-space: nowrap;
}

.qadisiyah-concept-switch__button {
  border: 1px solid rgba(47, 125, 50, 0.16);
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.86);
  color: #2f3a2f;
  cursor: pointer;
  font-size: 10px;
  font-weight: 800;
  line-height: 1;
  min-height: 24px;
  padding: 0 9px;
  transition:
    background 0.16s ease,
    color 0.16s ease,
    border-color 0.16s ease;
  white-space: nowrap;
}

.qadisiyah-concept-switch__button:hover,
.qadisiyah-concept-switch__button:focus-visible {
  border-color: rgba(47, 125, 50, 0.55);
  outline: none;
}

.qadisiyah-concept-switch__button.is-active {
  background: #2f7d32;
  border-color: #2f7d32;
  color: #ffffff;
}

.qadisiyah-concept-switch.is-arabic {
  direction: rtl;
}

.qadisiyah-concept-switch.is-dark-mode {
  background: rgba(44, 44, 44, 0.92);
  border-color: #555555;
}

.qadisiyah-concept-switch.is-dark-mode .qadisiyah-concept-switch__label,
.qadisiyah-concept-switch.is-dark-mode .qadisiyah-concept-switch__button {
  color: #ffffff;
}

.qadisiyah-concept-switch.is-dark-mode .qadisiyah-concept-switch__button {
  background: rgba(44, 44, 44, 0.86);
  border-color: #555555;
}

.qadisiyah-concept-switch.is-dark-mode
  .qadisiyah-concept-switch__button.is-active {
  background: #159447;
  border-color: #159447;
  color: #ffffff;
}

.main-map-style-switch__button {
  min-width: 72px;
  border: 0;
  border-right: 1px solid rgba(47, 125, 50, 0.15);
  background: transparent;
  color: #2f3a2f;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 12px;
  font-weight: 800;
  line-height: 1;
  padding: 0 13px;
  transition:
    background 0.16s ease,
    color 0.16s ease;
}

.main-map-style-switch.is-arabic .main-map-style-switch__button {
  min-width: 58px;
  padding: 0 8px;
  font-size: 11px;
  white-space: nowrap;
}

.main-map-style-switch.is-arabic
  .main-map-style-switch__button[data-map-mode="satellite"] {
  min-width: 74px;
}

.main-map-style-switch.is-arabic
  .main-map-style-switch__button[data-map-mode="3d"] {
  min-width: 52px;
}

.main-map-style-switch__button:last-child {
  border-right: 0;
}

.main-map-style-switch__button:hover,
.main-map-style-switch__button:focus-visible {
  background: rgba(47, 125, 50, 0.1);
  outline: none;
}

.main-map-style-switch__button.is-active {
  background: #2f7d32;
  color: #ffffff;
}

.main-map-style-switch.is-dark-mode {
  background: rgba(44, 44, 44, 0.94);
  border-color: #555555;
}

.main-map-style-switch.is-dark-mode .main-map-style-switch__button {
  border-right-color: #555555;
  color: #ffffff;
}

.main-map-style-switch.is-dark-mode .main-map-style-switch__button.is-active {
  background: #159447;
  color: #ffffff;
}

.main-map-layer-switch {
  position: fixed;
  top: var(--green-riyadh-map-filter-top, 18px);
  right: var(--green-riyadh-map-filter-right, 14px);
  z-index: 2147483646;
  display: flex;
  align-items: center;
  box-sizing: border-box;
  height: 42px;
  gap: 6px;
  padding: 0 8px;
  border-radius: 10px;
  background: rgba(255, 255, 255, 0.94);
  border: 1px solid rgba(0, 0, 0, 0.08);
  box-shadow: 0 6px 16px rgba(0, 0, 0, 0.12);
  pointer-events: auto;
  box-sizing: border-box;
}

.main-map-layer-switch.is-arabic {
  gap: 5px;
  padding: 0 7px;
  direction: rtl;
}

.main-map-layer-switch__label {
  color: #2f3a2f;
  font-size: 11px;
  font-weight: 700;
  white-space: nowrap;
}

.main-map-layer-switch__select {
  min-width: 132px;
  max-width: 150px;
  height: 30px;
  border: 1px solid rgba(47, 125, 50, 0.25);
  border-radius: 8px;
  background: #ffffff;
  color: #2f3a2f;
  cursor: pointer;
  font-size: 11px;
  font-weight: 700;
  line-height: 1;
  padding: 0 26px 0 9px;
  outline: none;
}

.main-map-layer-switch.is-arabic .main-map-layer-switch__select {
  min-width: 118px;
  max-width: 134px;
  padding: 0 9px 0 24px;
  text-align: right;
}

.main-map-layer-switch__select:hover,
.main-map-layer-switch__select:focus {
  border-color: #2f7d32;
}

.main-map-layer-switch.is-dark-mode {
  background: rgba(44, 44, 44, 0.94);
  border-color: #555555;
}

.main-map-layer-switch.is-dark-mode .main-map-layer-switch__label,
.main-map-layer-switch.is-dark-mode .main-map-layer-switch__select {
  color: #ffffff;
}

.main-map-layer-switch.is-dark-mode .main-map-layer-switch__select {
  background: #2c2c2c;
  border-color: #666666;
}

@media (max-width: 767px) {
  .three-park-compass {
    left: 14px;
    bottom: 30px;
    width: 78px;
    height: 78px;
  }

  .three-park-compass__north {
    top: 2px;
    font-size: 10px;
  }

  .three-park-compass__arrow {
    top: 15px;
    width: 29px;
    height: 38px;
  }

  .three-layer-filters {
    width: min(214px, var(--green-riyadh-map-layer-filter-width, 214px));
    padding: 10px;
  }

  .three-layer-filter-row {
    min-height: 24px;
    font-size: 10px;
  }

  .main-map-style-switch {
    left: var(--green-riyadh-map-control-left, 14px);
    right: auto;
    top: var(--green-riyadh-map-control-top, 14px);
  }

  .main-map-style-switch {
    height: 40px;
  }

  .qadisiyah-concept-switch {
    top: calc(var(--green-riyadh-map-control-top, 14px) + 48px);
    left: var(--green-riyadh-map-control-left, 14px);
    max-width: calc(100vw - var(--green-riyadh-map-control-left, 14px) - 22px);
    padding: 6px;
    gap: 5px;
  }

  .qadisiyah-concept-switch__label,
  .qadisiyah-concept-switch__button {
    font-size: 9px;
  }

  .qadisiyah-concept-switch__button {
    min-height: 23px;
    padding: 0 7px;
  }

  .main-map-style-switch__button {
    min-width: 62px;
    padding: 0 9px;
  }

  .main-map-style-switch.is-arabic .main-map-style-switch__button {
    min-width: 50px;
    padding: 0 6px;
    font-size: 10px;
  }

  .main-map-style-switch.is-arabic
    .main-map-style-switch__button[data-map-mode="satellite"] {
    min-width: 68px;
  }

  .main-map-style-switch.is-arabic
    .main-map-style-switch__button[data-map-mode="3d"] {
    min-width: 46px;
  }

  .main-map-layer-switch {
    left: auto;
    right: 12px;
    top: var(--green-riyadh-map-filter-top, 14px);
    height: 40px;
    padding: 0 6px;
  }

  .main-map-layer-switch__label {
    font-size: 10px;
  }

  .main-map-layer-switch__select {
    min-width: 112px;
    max-width: 124px;
    height: 28px;
    font-size: 10px;
    padding: 0 22px 0 7px;
  }

  .main-map-layer-switch.is-arabic .main-map-layer-switch__select {
    min-width: 104px;
    max-width: 116px;
    padding: 0 7px 0 20px;
  }
}

#map {
  background: gray;
  position: fixed !important;
  top: 0;
  left: 0;
  width: 32%;
  height: 100%;
}

#map.wide {
  width: 64%;
}

.map-navigation-controls {
  position: fixed !important;
  right: var(--green-riyadh-map-filter-right, 14px) !important;
  left: auto !important;
  bottom: var(--green-riyadh-map-nav-bottom, 70px) !important;
  z-index: 2147483646 !important;
  display: flex !important;
  visibility: visible !important;
  opacity: 1 !important;
  flex-direction: column;
  width: 40px;
  min-width: 40px;
  border-radius: 4px;
  overflow: hidden;
  background: #ffffff !important;
  box-shadow: 0 1px 5px rgba(0, 0, 0, 0.38);
  pointer-events: auto !important;
}

#map.wide ~ .map-navigation-controls {
  right: 14px !important;
  left: auto !important;
}

.map-navigation-button {
  width: 40px;
  height: 40px;
  padding: 0;
  border: 0;
  border-bottom: 1px solid #e1e1e1;
  background: #ffffff;
  color: #3c4043;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  font-family: Arial, sans-serif;
  line-height: 1;
  user-select: none;
}

.map-navigation-button:last-child {
  border-bottom: 0;
}

.map-navigation-button:hover,
.map-navigation-button:focus-visible {
  background: #f4f4f4;
  outline: none;
}

.map-zoom-button {
  font-size: 28px;
  font-weight: 300;
}

.map-fullscreen-button {
  border-bottom: 0;
}

.map-fullscreen-icon {
  width: 21px;
  height: 21px;
  fill: none;
  stroke: currentColor;
  stroke-width: 2;
  stroke-linecap: square;
  stroke-linejoin: miter;
}

.map-fullscreen-button.is-active {
  background: #f4f4f4;
}

.map-reset-button {
  font-size: 0;
}

.map-reset-icon {
  display: block;
  font-size: 24px;
  font-weight: 700;
  line-height: 1;
  transform: translateY(-1px);
}

.map-location-arrow {
  display: block;
  width: 0;
  height: 0;
  border-left: 8px solid transparent;
  border-right: 8px solid transparent;
  border-bottom: 19px solid #159447;
  transform: rotate(42deg);
  filter: drop-shadow(0 1px 0 rgba(0, 0, 0, 0.18));
}

.mapbox-user-location-marker {
  width: 34px;
  height: 34px;
  display: flex;
  align-items: center;
  justify-content: center;
  pointer-events: none;
}

.mapbox-user-location-arrow {
  display: block;
  width: 0;
  height: 0;
  border-left: 9px solid transparent;
  border-right: 9px solid transparent;
  border-bottom: 22px solid #159447;
  transform: rotate(42deg);
  filter: drop-shadow(0 1px 0 rgba(0, 0, 0, 0.26));
}

.map-location-button.is-locating .map-location-arrow {
  animation: map-location-pulse 0.9s ease-in-out infinite alternate;
}

.map-location-button.has-location {
  background: #edf8f0;
}

.map-location-message {
  position: fixed;
  left: calc(32% - 300px);
  bottom: 202px;
  z-index: 10002;
  max-width: 275px;
  padding: 9px 12px;
  border-radius: 6px;
  background: rgba(35, 35, 35, 0.92);
  color: #ffffff;
  font-size: 12px;
  line-height: 1.35;
  box-shadow: 0 4px 14px rgba(0, 0, 0, 0.2);
}

#map.wide ~ .map-location-message {
  left: calc(64% - 300px);
}

@keyframes map-location-pulse {
  from {
    opacity: 0.45;
    transform: rotate(42deg) scale(0.86);
  }
  to {
    opacity: 1;
    transform: rotate(42deg) scale(1);
  }
}

@media (max-width: 767px) {
  .map-navigation-controls,
  #map.wide ~ .map-navigation-controls {
    left: auto !important;
    right: var(--green-riyadh-map-filter-right, 12px) !important;
    bottom: var(--green-riyadh-map-nav-bottom, 72px) !important;
  }

  .map-location-message {
    left: 12px;
    right: 64px;
    bottom: 202px;
    max-width: none;
  }
}

.tse-map-legend {
  background: rgba(255, 255, 255, 0.86);
  border: 1px solid rgba(31, 111, 209, 0.08);
  border-radius: 8px;
  box-shadow: 0 6px 18px rgba(0, 0, 0, 0.1);
  color: #2f3a2f;
  font-size: 11px;
  line-height: 1.15;
  margin: 0 0 14px 12px;
  padding: 8px 10px;
  backdrop-filter: blur(6px);
}

.tse-map-legend__title {
  font-weight: 700;
  margin-bottom: 6px;
}

.tse-map-legend__row {
  align-items: center;
  display: flex;
  gap: 7px;
  margin: 3px 0;
}

.tse-map-legend__row--clickable {
  background: transparent;
  border: 0;
  color: inherit;
  cursor: pointer;
  font: inherit;
  padding: 2px 0;
  text-align: left;
  width: 100%;
}

.tse-map-legend__row--clickable:hover {
  opacity: 0.82;
}

.tse-map-legend__row--clickable.is-disabled {
  color: #8f9690;
}

.tse-map-legend__line {
  border-radius: 999px;
  display: inline-block;
  height: 6px;
  width: 30px;
  flex: 0 0 30px;
}

.tse-map-legend__line.is-muted {
  background: #b8b8b8 !important;
  opacity: 0.7;
}

.tse-map-legend--overlay {
  position: fixed;
  bottom: 18px;
  right: calc(32vw - 150px);
  z-index: 9999;
  min-width: 130px;
  background: rgba(255, 255, 255, 0.84);
  border: 1px solid rgba(31, 111, 209, 0.08);
  border-radius: 10px;
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.12);
  color: #2f3a2f;
  font-size: 12px;
  line-height: 1.2;
  padding: 9px 12px;
  backdrop-filter: blur(6px);
}

@media (max-width: 767px) {
  .tse-map-legend--overlay {
    left: 16px;
    right: auto;
    bottom: 16px;
  }
}

#map.wide + .tse-map-legend--overlay {
  right: calc(64vw - 150px);
}

.tse-map-legend__triangle {
  width: 0;
  height: 0;
  border-left: 5px solid transparent;
  border-right: 5px solid transparent;
  border-bottom: 8px solid #ffd700;
  display: inline-block;
  margin-right: 6px;
}

.tse-map-legend__line.is-default-completed {
  background: #61d76c;
}

.tse-map-legend__line.is-default-progress {
  background: #ffae00;
}

.tse-map-legend__line.is-tse-completed {
  background: #1f7a3a;
}

.tse-map-legend__line.is-tse-progress {
  background: #1f6fd1;
}

.mapboxgl-popup-content {
  border-radius: 8px;
  padding: 10px 12px;
  box-shadow: 0 8px 22px rgba(0, 0, 0, 0.2);
}

.mapboxgl-popup-tip {
  opacity: 0.95;
}

/* Mapbox popups are created outside Vue's scoped DOM, so give the popup
   its own dark-mode class based on this component's `theme` value. */
:global(.mapboxgl-popup.is-dark-mode .mapboxgl-popup-content) {
  background: #2c2c2c !important;
  color: #ffffff !important;
  border: 1px solid #555555 !important;
  box-shadow: 0 8px 22px rgba(0, 0, 0, 0.45) !important;
}

:global(.mapboxgl-popup.is-dark-mode .marker-tool),
:global(.mapboxgl-popup.is-dark-mode .marker-tool *) {
  color: #ffffff !important;
}

:global(
  .mapboxgl-popup.is-dark-mode.mapboxgl-popup-anchor-top .mapboxgl-popup-tip
) {
  border-bottom-color: #2c2c2c !important;
}

:global(
  .mapboxgl-popup.is-dark-mode.mapboxgl-popup-anchor-bottom .mapboxgl-popup-tip
) {
  border-top-color: #2c2c2c !important;
}

:global(
  .mapboxgl-popup.is-dark-mode.mapboxgl-popup-anchor-left .mapboxgl-popup-tip
) {
  border-right-color: #2c2c2c !important;
}

:global(
  .mapboxgl-popup.is-dark-mode.mapboxgl-popup-anchor-right .mapboxgl-popup-tip
) {
  border-left-color: #2c2c2c !important;
}

:global(
  .mapboxgl-popup.is-dark-mode.mapboxgl-popup-anchor-top-left
    .mapboxgl-popup-tip
),
:global(
  .mapboxgl-popup.is-dark-mode.mapboxgl-popup-anchor-top-right
    .mapboxgl-popup-tip
) {
  border-bottom-color: #2c2c2c !important;
}

:global(
  .mapboxgl-popup.is-dark-mode.mapboxgl-popup-anchor-bottom-left
    .mapboxgl-popup-tip
),
:global(
  .mapboxgl-popup.is-dark-mode.mapboxgl-popup-anchor-bottom-right
    .mapboxgl-popup-tip
) {
  border-top-color: #2c2c2c !important;
}

:global(html.dark) .marker-tool,
:global(html.dark) .marker-tool *,
:global(body.dark) .marker-tool,
:global(body.dark) .marker-tool *,
:global(.dark) .marker-tool,
:global(.dark) .marker-tool *,
:global(.dark-mode) .marker-tool,
:global(.dark-mode) .marker-tool *,
:global([data-theme="dark"]) .marker-tool,
:global([data-theme="dark"]) .marker-tool * {
  color: #ffffff !important;
}

:global(html.dark) .mapboxgl-popup-content,
:global(body.dark) .mapboxgl-popup-content,
:global(.dark) .mapboxgl-popup-content,
:global(.dark-mode) .mapboxgl-popup-content,
:global([data-theme="dark"]) .mapboxgl-popup-content {
  background: #2c2c2c !important;
  color: #ffffff !important;
}

:global(html.dark) .tse-map-legend,
:global(body.dark) .tse-map-legend,
:global(.dark) .tse-map-legend,
:global(.dark-mode) .tse-map-legend,
:global([data-theme="dark"]) .tse-map-legend {
  background: rgba(35, 35, 35, 0.84) !important;
  border-color: rgba(255, 255, 255, 0.1) !important;
  color: #ffffff !important;
}

.dark .tse-map-legend {
  background: #2c2c2c !important;
  color: #ffffff !important;
  border: 1px solid #444 !important;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.4);
}
.dark .tse-map-legend__row--clickable.is-disabled,
.dark .tse-map-legend__row--clickable.is-disabled span {
  color: #8b9491 !important;
}

.dark .tse-map-legend div,
.dark .tse-map-legend span,
.dark .tse-map-legend p {
  color: #ffffff !important;
}

:global(html.dark) .map-navigation-controls,
:global(body.dark) .map-navigation-controls,
:global(.dark) .map-navigation-controls,
:global(.dark-mode) .map-navigation-controls,
:global([data-theme="dark"]) .map-navigation-controls {
  background: #2c2c2c !important;
  border: 1px solid #444444 !important;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.4) !important;
}

:global(html.dark) .map-navigation-button,
:global(body.dark) .map-navigation-button,
:global(.dark) .map-navigation-button,
:global(.dark-mode) .map-navigation-button,
:global([data-theme="dark"]) .map-navigation-button {
  background: #2c2c2c !important;
  color: #ffffff !important;
  border-bottom-color: #444444 !important;
}

:global(html.dark) .map-navigation-button:hover,
:global(html.dark) .map-navigation-button:focus-visible,
:global(body.dark) .map-navigation-button:hover,
:global(body.dark) .map-navigation-button:focus-visible,
:global(.dark) .map-navigation-button:hover,
:global(.dark) .map-navigation-button:focus-visible,
:global(.dark-mode) .map-navigation-button:hover,
:global(.dark-mode) .map-navigation-button:focus-visible,
:global([data-theme="dark"]) .map-navigation-button:hover,
:global([data-theme="dark"]) .map-navigation-button:focus-visible {
  background: #3a3a3a !important;
}

:global(html.dark) .map-location-button.has-location,
:global(body.dark) .map-location-button.has-location,
:global(.dark) .map-location-button.has-location,
:global(.dark-mode) .map-location-button.has-location,
:global([data-theme="dark"]) .map-location-button.has-location {
  background: #263b2d !important;
}

.map-navigation-controls.is-dark-mode {
  background: rgba(44, 44, 44, 0.94) !important;
  border-color: #555555 !important;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.45) !important;
}

.map-navigation-controls.is-dark-mode .map-navigation-button {
  background: rgba(44, 44, 44, 0.94) !important;
  color: #ffffff !important;
  border-bottom-color: #555555 !important;
}

.map-navigation-controls.is-dark-mode .map-navigation-button:hover,
.map-navigation-controls.is-dark-mode .map-navigation-button:focus-visible {
  background: #3b3b3b !important;
}

.map-navigation-controls.is-dark-mode .map-location-button.has-location {
  background: #263b2d !important;
}

#map .mapboxgl-canvas.is-middle-mouse-panning {
  cursor: grabbing !important;
}

/* Layer Filters: remove the small decorative square to the LEFT of labels.
   Keep right-side checkboxes and expand/collapse arrows unchanged. */
.three-layer-filters .layer-filter-label::before,
.three-layer-filters .layer-filter-name::before,
.three-layer-filters .layer-filter-row > .layer-filter-label::before,
.three-layer-filters .three-layer-filter-label::before,
.three-layer-filters .three-layer-filter-name::before,
.three-layer-filters
  .three-layer-filter-row
  > .three-layer-filter-label::before,
.three-layer-filters .filter-label::before,
.three-layer-filters .filter-name::before,
.layer-filters-panel .layer-filter-label::before,
.layer-filters-panel .layer-filter-name::before,
.layer-filters-panel .filter-label::before,
.layer-filters-panel .filter-name::before {
  display: none !important;
  content: none !important;
  width: 0 !important;
  height: 0 !important;
  margin: 0 !important;
  border: 0 !important;
}

/* Qadisiyah 3D UI theme ----------------------------------------------------
   Match the lime-green concept styling while leaving GIS/model materials
   unchanged. This is intentionally appended so it wins over older cyan UI
   rules without changing rendering logic. */
.map-component-wrap {
  --qad-ui-green: #86dc4f;
  --qad-ui-green-bright: #99ea61;
  --qad-ui-green-soft: rgba(134, 220, 79, 0.38);
  --qad-ui-green-faint: rgba(134, 220, 79, 0.16);
  --qad-ui-panel: rgba(8, 17, 14, 0.92);
  --qad-ui-panel-soft: rgba(8, 17, 14, 0.78);
}

/* View selector */
.main-map-style-switch__button:hover,
.main-map-style-switch__button:focus-visible {
  background: rgba(134, 220, 79, 0.12);
}

.main-map-style-switch__button.is-active,
.main-map-style-switch.is-dark-mode .main-map-style-switch__button.is-active {
  background: var(--qad-ui-green) !important;
  color: #10200d !important;
}

/* Layer Filters */
.three-layer-filters {
  border-color: var(--qad-ui-green-soft);
  background: var(--qad-ui-panel);
}

.three-layer-filters__title,
.three-layer-filters__visibility-column-title,
.three-layer-filters__label-column-title,
.three-layer-filter-row__arrow {
  color: var(--qad-ui-green-bright);
}

.three-layer-filter-row__checkbox {
  accent-color: var(--qad-ui-green);
}

.three-layer-filter-row.is-partial .three-layer-filter-row__icon {
  background: var(--qad-ui-green-soft);
}

.three-layer-filter-children,
.three-layer-filters.is-arabic .three-layer-filter-children {
  border-color: rgba(134, 220, 79, 0.28);
}

/* Left 3D tool strip */
.three-park-tool-panel {
  border-color: rgba(134, 220, 79, 0.3);
  background: var(--qad-ui-panel-soft);
}

.three-park-tool-panel__button {
  border-color: rgba(134, 220, 79, 0.2);
  color: var(--qad-ui-green-bright);
}

.three-park-tool-panel__button:hover:not(.is-disabled),
.three-park-tool-panel__button:focus-visible:not(.is-disabled) {
  border-color: rgba(153, 234, 97, 0.72);
  background: rgba(53, 91, 35, 0.55);
  box-shadow: 0 0 0 1px rgba(134, 220, 79, 0.18);
}

.three-park-tool-panel__button.is-active {
  border-color: rgba(153, 234, 97, 0.88);
  background: rgba(63, 112, 39, 0.68);
  box-shadow:
    inset 0 0 12px rgba(134, 220, 79, 0.18),
    0 0 0 1px rgba(134, 220, 79, 0.22);
}

/* Structure labels: compact rectangular black cards with lime-green text. */
.three-park-label-overlay__item {
  padding: 6px 10px 5px;
  border-color: rgba(134, 220, 79, 0.42);
  border-radius: 6px;
  background: rgba(8, 13, 11, 0.94);
  color: var(--qad-ui-green-bright);
  box-shadow: 0 5px 14px rgba(0, 0, 0, 0.34);
  font-weight: 800;
}

.three-park-label-overlay__item::after {
  background: linear-gradient(
    to bottom,
    rgba(153, 234, 97, 0.9),
    rgba(153, 234, 97, 0)
  );
}

/* Compass */
.three-park-compass__dial {
  border-color: rgba(134, 220, 79, 0.54);
  background: radial-gradient(
    circle at center,
    rgba(7, 16, 12, 0.96) 0 42%,
    rgba(55, 100, 35, 0.58) 43% 45%,
    rgba(5, 13, 9, 0.94) 46% 61%,
    rgba(134, 220, 79, 0.58) 62% 64%,
    rgba(4, 11, 8, 0.88) 65% 100%
  );
  box-shadow:
    inset 0 0 0 4px rgba(20, 39, 15, 0.82),
    inset 0 0 18px rgba(134, 220, 79, 0.15),
    0 0 0 1px rgba(4, 12, 7, 0.9);
}

.three-park-compass__dial::before {
  border-color: rgba(134, 220, 79, 0.42);
  background: repeating-conic-gradient(
    from -0.5deg,
    rgba(153, 234, 97, 0.76) 0deg 1deg,
    transparent 1deg 29deg,
    rgba(84, 146, 52, 0.6) 29deg 30deg
  );
}

.three-park-compass__dial::after {
  border-color: rgba(134, 220, 79, 0.3);
  box-shadow: 0 0 8px rgba(134, 220, 79, 0.14);
}

.three-park-compass__north {
  color: var(--qad-ui-green-bright);
  text-shadow: 0 0 7px rgba(134, 220, 79, 0.56);
}

.three-park-compass__arrow {
  filter: drop-shadow(0 0 5px rgba(134, 220, 79, 0.44));
}

.three-park-compass__arrow-main {
  fill: var(--qad-ui-green);
  stroke: rgba(191, 255, 145, 0.9);
}

.three-park-compass__arrow-cut {
  fill: rgba(11, 29, 8, 0.9);
}

.three-park-compass__center-dot {
  background: #b8f788;
  box-shadow: 0 0 7px rgba(153, 234, 97, 0.92);
}

/* Tree-type panel and loading state use the same accent family. */
.three-tree-type-panel {
  border-color: rgba(134, 220, 79, 0.34);
  background: var(--qad-ui-panel);
}

.three-tree-type-panel__header,
.three-tree-type-panel__row {
  border-color: rgba(134, 220, 79, 0.14);
}

.three-tree-type-panel__total strong,
.three-tree-type-panel__count {
  color: var(--qad-ui-green-bright);
}

.three-tree-type-panel__count {
  background: rgba(62, 110, 39, 0.46);
}

.three-tree-type-panel__close:hover,
.three-tree-type-panel__close:focus-visible {
  border-color: rgba(153, 234, 97, 0.72);
  background: rgba(55, 100, 35, 0.62);
}

.three-park-status {
  border-color: rgba(134, 220, 79, 0.48);
  color: #e8ffe0;
}
</style>
