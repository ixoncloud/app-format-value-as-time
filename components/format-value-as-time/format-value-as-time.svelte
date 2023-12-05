<script lang="ts">
  import { onMount } from 'svelte';
  import type { ComponentContext } from '@ixon-cdk/types';
  import { durationToformattedTimeStamp } from './utils/format';
  import { mapMetricInputToQuery } from './utils/query';

  export let context: ComponentContext;

  let rootEl: HTMLDivElement;

  // Data
  let height: number | null = null;
  let calculatedValue: number | null = null;
  let text = '';
  let error = '';
  let textColor = 'black';

  // Computed properties
  $: hasHeader = header && (header.title || header.subtitle) && !isShallow;
  $: isShallow = height !== null ? height <= 60 : false;
  $: header = context ? context.inputs.header : undefined;
  $: scaledTextStyle = `fill: ${textColor || 'inherit'}`;

  const cardContentTextStyle = 'color: ' + textColor;

  $: hasStaticSize = _hasStaticSize(context);
  function _hasStaticSize(context: ComponentContext) {
    if (context && context.inputs && context.inputs.style) {
      return context.inputs.style.fontSize !== 'auto';
    }
    return false;
  }

  $: staticSizeStyle = _staticSizeStyle(hasStaticSize);
  function _staticSizeStyle(hasStaticSize: boolean) {
    if (hasStaticSize) {
      return `font-size: ${context.inputs.style.fontSize}px;`;
    }
    return null;
  }

  $: svgViewBox = _svgViewBox(text);
  function _svgViewBox(text: string) {
    const textLength = text.length - (text.startsWith('-') ? 1 : 0);
    const fontSize = 14.0;
    const textWidth = textLength * fontSize * 0.6;
    const textHeight = fontSize;
    return `${-textWidth / 2} ${-textHeight / 2} ${textWidth} ${textHeight}`;
  }

  // Events
  onMount(() => {
    context.ontimerangechange = () => {
      calculatedValue = null;
      text = '';
      error = '';
    };

    const client = context.createLoggingDataClient();

    const query = {
      ...mapMetricInputToQuery(context.inputs.dataSource.metric),
      limit: 1,
    };

    const cancelQuery = client.query(query, metrics => {
      const metricValue = metrics.length && metrics[0].value.getValue();
      const isNum = typeof metricValue === 'number';
      if (isNum) {
        calculatedValue = metricValue;
        text = durationToformattedTimeStamp(metricValue);
      } else {
        error = 'NaN';
      }
    });

    const resizeObserver = new ResizeObserver(entries => {
      entries.forEach(entry => {
        height = entry.contentRect.height;
      });
    });
    resizeObserver.observe(rootEl);

    return () => {
      if (cancelQuery) {
        cancelQuery();
      }
      context.ontimerangechange = null;
      resizeObserver.unobserve(rootEl);
      client.destroy();
    };
  });
</script>

<div class="card" bind:this={rootEl}>
  {#if hasHeader}
    <div class="card-header">
      {#if header.title}
        <h3 class="card-title">{header.title}</h3>
      {/if}
      {#if header.subtitle}
        <h4 class="card-subtitle">{header.subtitle}</h4>
      {/if}
    </div>
  {/if}
  {#if text !== null}
    <div
      class="card-content"
      class:has-header={hasHeader}
      style={cardContentTextStyle}
    >
      {#if error}
        <div class="static" style={staticSizeStyle}>
          <p>{error}</p>
        </div>
      {/if}
      {#if hasStaticSize}
        <div class="static" style={staticSizeStyle}>
          <span>{text}</span>
        </div>
      {:else}
        <div class="scaled">
          <svg viewBox={svgViewBox}>
            <text
              x="0"
              y="0"
              text-anchor="middle"
              dominant-baseline="middle"
              style={scaledTextStyle}>{text}</text
            >
          </svg>
        </div>
      {/if}
    </div>
  {/if}
</div>

<style lang="scss">
  @import './styles/card';

  .card .card-content {
    height: 100%;

    &.has-header {
      height: calc(100% - 40px);
    }

    .static {
      height: 100%;
      display: flex;
      flex-direction: row;
      place-content: center;
      align-items: center;
      flex: 1;

      span {
        white-space: nowrap;
      }
    }

    .scaled {
      height: 100%;
      display: flex;
      justify-content: center;

      svg {
        width: 100%;

        text {
          font-size: 14px;
          fill: var(--body-color);
        }
      }
    }
  }
</style>
