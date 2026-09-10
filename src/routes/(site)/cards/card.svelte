<script lang="ts">
  import { base } from '$app/paths';
  import '@picocss/pico/css/pico.css';
  import { locations, type CharacterName } from '../../../model/characters';
  import Iron from './iron.svelte';
  import { Spring, spring } from 'svelte/motion';
  import { adjust, clamp, round } from '../../../misc';
  import { getString } from '../+layout.svelte';
  import { enableTranslationUi } from '../../+layout.svelte';
  import type { PageServerData } from './$types';
  import './holo.css';
  import { getAvialableCharacterImages } from '../../+layout.svelte';
  import Translation from '../../../view/translation.svelte';

  import * as data from '../../../data';

  const characterImages = getAvialableCharacterImages();

  type Card = {
    name: string;
    type: 'character';
    edition?: string;
    startLocation: readonly (typeof locations)[number][] | undefined;
    forbiddenLocation: readonly (typeof locations)[number][];
    gender: 'male' | 'female' | 'both' | 'diverse';
    paranoiaLimit: number | undefined;
    image: string;
    tags: readonly string[];
    abilities: (
      | {
          type: 'passive';
          description: string;
          restrictedToLocation: readonly string[];
        }
      | {
          type: 'active';
          description: string;
          goodwillRank: number;
          timesPerLoop: number;
          immuneToGoodwillRefusal: boolean;
          restrictedToLocation: readonly string[];
        }
    )[];
  };

  let {
    scale = 1,
    animated = false,
    card,
    face = 'front',
  }: {
    scale?: number;
    animated?: boolean;
    card:
      | Card
      | CharacterName
      | ({
          type: 'character';
          key: CharacterName;
        } & Partial<Card>);
    face?: 'front' | 'back' | 'dead';
  } = $props();

  let actualCard = $derived(getCarddataFromName(card));

  function getCarddataFromName(c: typeof card): Card {
    const key =
      typeof c === 'string' ? c : 'key' in c ? c.key : 'id' in c ? c.id : (c.name as CharacterName);
    const value =
      typeof c === 'string'
        ? data.charactersLookup[c]
        : 'key' in c
          ? data.charactersLookup[c.key]
          : c;
    if (value === undefined) {
      throw new Error(`Character with name ${key} not found`);
    }
    const dataFromCharacterName = {
      type: 'character' as const,
      ...value,
      forbiddenLocation:
        'forbiddenLocation' in value && value.forbiddenLocation ? value.forbiddenLocation : [],
      name: `:${key}:`,
      gender:
        (value.tags.includes('boy' as never) || value.tags.includes('man' as never)) &&
        (value.tags.includes('girl' as never) || value.tags.includes('woman' as never))
          ? ('both' as const)
          : value.tags.includes('boy' as never) || value.tags.includes('man' as never)
            ? ('male' as const)
            : value.tags.includes('girl' as never) || value.tags.includes('woman' as never)
              ? ('female' as const)
              : ('diverse' as const),
      tags: value.tags.toSorted((a, b) =>
        $getString(data.keywordsLookup[a as keyof typeof data.keywordsLookup].name).localeCompare(
          $getString(data.keywordsLookup[b as keyof typeof data.keywordsLookup].name)
        )
      ),
      image: characterImages[key as keyof typeof characterImages],
      abilities:
        value.abilities?.map((ability) => {
          return {
            ...ability,
            timesPerLoop:
              'timesPerLoop' in ability && ability.timesPerLoop != undefined
                ? ability.timesPerLoop
                : 0,
            immuneToGoodwillRefusal:
              'immuneToGoodwillRefusal' in ability && ability.immuneToGoodwillRefusal !== undefined
                ? ability.immuneToGoodwillRefusal
                : false,
            restrictedToLocation:
              'restrictedToLocation' in ability && ability.restrictedToLocation
                ? ability.restrictedToLocation
                : [],

            description: ability.description,
          };
        }) ?? [],
    };

    if (typeof c === 'string') {
      return dataFromCharacterName;
    } else {
      return {
        ...dataFromCharacterName,
        ...c,
      };
    }
  }

  const springInteractSettings = { stiffness: 0.066, damping: 0.25 };
  const springPopoverSettings = { stiffness: 0.033, damping: 0.45 };

  let springRotate = new Spring({ x: 0, y: 0 }, springInteractSettings);
  let springGlare = new Spring({ x: 50, y: 50, o: 0 }, springInteractSettings);
  let springBackground = new Spring({ x: 50, y: 50 }, springInteractSettings);
  let springRotateDelta = new Spring({ x: 0, y: 0 }, springPopoverSettings);
  let springTranslate = new Spring({ x: 0, y: 0 }, springPopoverSettings);
  let springScale = new Spring(1, springPopoverSettings);

  const updateSprings = (
    background: { x: number; y: number },
    rotate: { x: number; y: number },
    glare: { x: number; y: number; o: number }
  ) => {
    springBackground.stiffness = springInteractSettings.stiffness;
    springBackground.damping = springInteractSettings.damping;
    springRotate.stiffness = springInteractSettings.stiffness;
    springRotate.damping = springInteractSettings.damping;
    springGlare.stiffness = springInteractSettings.stiffness;
    springGlare.damping = springInteractSettings.damping;

    springBackground.set(background);
    springRotate.set(rotate);
    springGlare.set(glare);
  };

  const interact = (e: PointerEvent & { currentTarget: EventTarget & HTMLDivElement }) => {
    if (!animated) return;
    // endShowcase();

    const el = e.currentTarget;
    const rect = el?.getBoundingClientRect(); // get element's current size/position
    const absolute = {
      x: e.clientX - rect.left, // get mouse position from left
      y: e.clientY - rect.top, // get mouse position from right
    };
    const percent = {
      x: clamp(round((100 / rect.width) * absolute.x)),
      y: clamp(round((100 / rect.height) * absolute.y)),
    };
    const center = {
      x: percent.x - 50,
      y: percent.y - 50,
    };

    updateSprings(
      {
        x: adjust(percent.x, 0, 100, 37, 63),
        y: adjust(percent.y, 0, 100, 33, 67),
      },
      {
        x: round(-(center.x / 3.5)),
        y: round(center.y / 2),
      },
      {
        x: round(percent.x),
        y: round(percent.y),
        o: 1,
      }
    );
  };

  const interactEnd = (e: unknown, delay = 500) => {
    if (!animated) return;

    setTimeout(function () {
      const snapStiff = 0.01;
      const snapDamp = 0.06;

      springRotate.stiffness = snapStiff;
      springRotate.damping = snapDamp;
      springRotate.set({ x: 0, y: 0 }, { soft: 1 });

      springGlare.stiffness = snapStiff;
      springGlare.damping = snapDamp;
      springGlare.set({ x: 50, y: 50, o: 0 }, { soft: 1 });

      springBackground.stiffness = snapStiff;
      springBackground.damping = snapDamp;
      springBackground.set({ x: 50, y: 50 }, { soft: 1 });
    }, delay);
  };

  let dynamicStyles = $derived(
    animated
      ? `
    --pointer-x: ${springGlare.current.x}%;
    --pointer-y: ${springGlare.current.y}%;
    --pointer-from-center: ${clamp(
      Math.sqrt(
        (springGlare.current.y - 50) * (springGlare.current.y - 50) +
          (springGlare.current.x - 50) * (springGlare.current.x - 50)
      ) / 50,
      0,
      1
    )};
    --pointer-from-top: ${springGlare.current.y / 100};
    --pointer-from-left: ${springGlare.current.x / 100};
    --card-opacity: ${springGlare.current.o};
    --rotate-x: ${springRotate.current.x + springRotateDelta.current.x}deg;
    --rotate-y: ${springRotate.current.y + springRotateDelta.current.y}deg;
    --background-x: ${springBackground.current.x}%;
    --background-y: ${springBackground.current.y}%;
    --card-scale: ${springScale};
    --translate-x: ${springTranslate.current.x}px;
    --translate-y: ${springTranslate.current.y}px;
    --scale:${scale};
	`
      : ''
  );
</script>

{#if card}
  <div class:card__holder={animated} style={dynamicStyles} class="top">
    <div class="card__translater">
      <!-- svelte-ignore a11y_no_static_element_interactions -->
      <div
        class="card__rotator"
        onpointermove={interact}
        onmouseout={interactEnd}
        onblur={interactEnd}
      >
        <div
          class="card"
          class:blood={face == 'dead'}
          data-number="132"
          data-set="swsh9"
          data-subtypes="supporter"
          data-supertype="trainer"
          data-rarity="rare holo"
          data-trainer-gallery="false"
          style="--scale:{scale};"
        >
          {#if face != 'back'}
            <img src="{base}/cards/general/background.png" alt="Character" class="back" />
            <img src="{base}/cards/general/blood.png" alt="Character" class="back splatter" />
            <div class="card__shine back transformable"></div>
            <div class="card__glare back transformable"></div>

            {#if actualCard.image}
              <img src={actualCard.image} alt="Character" class="back image character" />

              <div class="back glow">
                <div
                  data-clip={actualCard.image}
                  style="--mask: url('{actualCard.image}');"
                  class="bitten outer back"
                >
                  <div class="blurred back">
                    <div data-clip={actualCard.image} class="back glow bitten"></div>
                    <!-- <div class="bitten"></div> -->
                  </div>
                </div>
              </div>
            {/if}
            <img
              src="{base}/cards/general/diverse.png"
              alt="Cardbackground"
              class="back"
            />

            <div class="location-background-grid">
              {#each Array.from({ length: 4 }) as _, idx}
                <img
                  src="{base}/cards/general/location-background.png"
                  alt=""
                  class="location-background-tile"
                />
              {/each}
            </div>

            {#each locations.filter((x) => x !== 'The Far Side') as location}
              {#if actualCard.startLocation?.includes(location)}
                {#if location === 'Hospital'}
                  <img
                    src="{base}/cards/general/hospital-icon.svg"
                    alt={$getString('Hospital')}
                    class="location-background-icon icon-hospital"
                  />
                {/if}
                {#if location === 'Shrine'}
                  <img
                    src="{base}/cards/general/shrine-icon.svg"
                    alt={$getString('Shrine')}
                    class="location-background-icon icon-shrine"
                  />
                {/if}
                {#if location === 'City'}
                  <img
                    src="{base}/cards/general/city-icon.svg"
                    alt={$getString('City')}
                    class="location-background-icon icon-city"
                  />
                {/if}
                {#if location === 'School'}
                  <img
                    src="{base}/cards/general/school-icon.svg"
                    alt={$getString('School')}
                    class="location-background-icon icon-school"
                  />
                {/if}
              {:else if actualCard.forbiddenLocation.includes(location)}
                <img
                  src="{base}/icons/custom/scar.png"
                  alt=""
                  class="location-background-forbidden forbidden-{location.toLocaleLowerCase()}"
                />
              {/if}
            {/each}

            
            {#if actualCard.paranoiaLimit !== undefined}
              <ul class="paranoia">
                {#each Array.from({ length: actualCard.paranoiaLimit }) as _, i}
                  <li>
                    <img src="{base}/icons/custom/paranoia2_custom.png" alt="paranoia icon" />
                  </li>
                {/each}
              </ul>
              <div class="paranoia-limit-bar">
                <span>paranoia limit</span>
              </div>
              <div class="paranoia-limit-number">
                <span>{actualCard.paranoiaLimit}</span>
              </div>
            {/if}

            <h2><Translation translationKey={actualCard.name} /></h2>

            <ul class="abilities">
              {#each actualCard.abilities as ability}
                <li class={ability.type}>
                  {#if ability.type == 'active'}
                    <div class="ability-icons">
                      {#if ability.timesPerLoop > 0}
                        <ul class="perLoop">
                          <li>
                            <span class="perLoopText">
                              {ability.timesPerLoop} × <span class="perLoopInfinity">∞</span>
                            </span>
                          </li>
                        </ul>
                      {/if}
                      <ul class="goodwillRank">
                        {#each Array.from({ length: ability.goodwillRank }) as _, i}
                          <li>
                            <img src="{base}/cards/general/goodwill_custom.png" alt="goodwill icon" />
                          </li>
                        {/each}
                      </ul>
                    </div>
                  {/if}
                  <div class="ability">
                    {#if ability.restrictedToLocation.length > 0}
                      <div>
                        <Translation translationKey="Only at" />: {#each ability.restrictedToLocation as res, i}
                          <Translation translationKey={res} />
                          {#if i < ability.restrictedToLocation.length - 1},
                          {/if}
                        {/each}
                      </div>
                    {/if}
                    <Translation translationKey={ability.description} />
                  </div>
                </li>
              {/each}
            </ul>
            {#if enableTranslationUi.showEdition && actualCard.edition}
              <div class="card-edition">
                <span class="edition-value">{actualCard.edition}</span>
              </div>
            {/if}
            <ul class="tags">
              {#each actualCard.tags as tag}
                <li>
                  <!-- <Iron /> -->
                  <div>
                    <Translation
                      translationKey={data.keywordsLookup[tag as keyof typeof data.keywordsLookup]
                        .name}
                    />
                  </div>
                  <!-- <Iron /> -->
                </li>
              {/each}
            </ul>
          {:else}
            <img src="{base}/cards/general/cardback.png" alt="Empty" class="back" />
            <div class="card__shine back transformable"></div>
            <div class="card__glare back transformable"></div>
          {/if}
        </div>
      </div>
    </div>
  </div>
{/if}

<style lang="scss">
  @import url('https://fonts.googleapis.com/css2?family=Nanum+Gothic:wght@400;700&display=swap');

  .top {
    height: calc(8.8cm * var(--scale, 1));
    width: calc(6.3cm * var(--scale, 1));
  }
  .card {
    box-sizing: border-box;
    break-inside: avoid;
    height: calc(8.8cm * var(--scale, 1));
    width: calc(6.3cm * var(--scale, 1));
    position: relative;
    border: calc(3px * var(--scale)) solid #000000;
    border-radius: calc(0.2cm * var(--scale));
    background-color: #000;
    font-family: 'Noto Sans';

    @media print {
      break-inside: avoid;
      page-break-inside: avoid;
      border-color: black;
      //   box-shadow: #000 0 0 0cm 0.3cm;
      // margin: 3.31cm;
    }

    padding: 0;
    overflow: hidden;

    font-size: calc(8pt * var(--scale));

    .splatter {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      z-index: -1;
      pointer-events: none;
    }
    .splatter,
    .glow {
      display: none;
    }
    &.blood {
      .image {
        display: none;
      }

      .splatter {
        z-index: 0;
      }

      .splatter,
      .glow {
        display: block;
      }

      .blurred {
        filter: blur(calc(8px * var(--scale)));
      }

      .bitten {
        background: #4a8ad3b6;
        // clip-path: polygon(0 0, 0 100%, 80% 100%, 100% 0);
        mask-image: var(--mask);
        mask-size: contain;
        mask-repeat: no-repeat;
        mask-position: bottom center;
        margin-top: calc(0.2cm * var(--scale));
        &[data-clip*='tree'] {
          // hack for sacred tree
          // need to find a better way, but don't want to change every image
          mask-size: cover;
          margin: 0;
        }
      }

      .bitten.outer {
        position: relative;
        background: #ffffff;
        opacity: 0.5;
      }
    }

    .image {
      filter: drop-shadow(0 0 calc(5px * var(--scale)) #b5add273);
    }
    .image {
      object-fit: contain;
      object-position: bottom center;
      padding-top: calc(0.2cm * var(--scale));
      // glow
      &[src*='tree'] {
        // hack for sacred tree
        // need to find a better way, but don't want to change every image
        object-fit: fill;
        padding: 0;
      }
    }
    .back {
      height: calc(8.8cm * var(--scale, 1));
      width: calc(6.3cm * var(--scale, 1));
      position: absolute;
    }
    .location {
      position: absolute;
      left: 0;
      top: 0;
      width: calc(6.3cm * var(--scale, 1));
      height: calc(8.8cm * var(--scale, 1));
      object-fit: cover;
      object-position: center;
      display: block;
      overflow: hidden;
      max-width: none;
      max-height: none;
    }
    .location-background-grid {
      position: absolute;
      top: calc(0.2cm * var(--scale));
      right: calc(0.2cm * var(--scale));
      width: calc(56px * var(--scale));
      height: calc(56px * var(--scale));
      display: grid;
      grid-template-columns: repeat(2, calc(28px * var(--scale)));
      grid-template-rows: repeat(2, calc(28px * var(--scale)));
      gap: 1px;
      z-index: 2;
      align-content: start;
      justify-content: end;
    }
    .location-background-tile {
      width: calc(28px * var(--scale));
      height: calc(28px * var(--scale));
      object-fit: cover;
      display: block;
      padding: 0;
      margin: 0;
    }
    .location-background-icon {
      position: absolute;
      top: calc(0.2cm * var(--scale));
      right: calc(0.2cm * var(--scale));
      width: calc(22px * var(--scale));
      height: calc(22px * var(--scale));
      object-fit: contain;
      display: block;
      padding: 0;
      margin: 0;
      z-index: 4;
      filter: grayscale(1) brightness(1.8) invert(1);
    }
    .location-background-forbidden {
      position: absolute;
      width: calc(24px * var(--scale));
      height: calc(24px * var(--scale));
      display: block;
      padding: 0;
      margin: 0;
      z-index: 5;
      object-fit: contain;
      object-position: center;
      filter: drop-shadow(0 0 calc(2px * var(--scale)) rgba(0, 0, 0, 0.95))
        drop-shadow(0 0 calc(3px * var(--scale)) rgba(255, 45, 45, 0.9));
      background: transparent;
      border: none;
    }
    .icon-hospital {
      top: calc(0.26cm * var(--scale));
      right: calc(0.26cm * var(--scale) + calc(28px * var(--scale)) + 1px);
    }
    .icon-shrine {
      top: calc(0.26cm * var(--scale));
      right: calc(0.26cm * var(--scale));
    }
    .icon-city {
      top: calc(0.26cm * var(--scale) + calc(28px * var(--scale)) + 1px);
      right: calc(0.26cm * var(--scale) + calc(28px * var(--scale)) + 1px);
    }
    .icon-school {
      top: calc(0.26cm * var(--scale) + calc(28px * var(--scale)) + 1px);
      right: calc(0.26cm * var(--scale));
    }
    .forbidden-hospital {
      top: calc(0.26cm * var(--scale));
      right: calc(0.26cm * var(--scale) + calc(28px * var(--scale)) + 1px);
    }
    .forbidden-shrine {
      top: calc(0.26cm * var(--scale));
      right: calc(0.26cm * var(--scale));
    }
    .forbidden-city {
      top: calc(0.26cm * var(--scale) + calc(28px * var(--scale)) + 1px);
      right: calc(0.26cm * var(--scale) + calc(28px * var(--scale)) + 1px);
    }
    .forbidden-school {
      top: calc(0.26cm * var(--scale) + calc(28px * var(--scale)) + 1px);
      right: calc(0.26cm * var(--scale));
    }
    h2 {
      position: absolute;
      display: flex;
      top: calc(1.6cm * var(--scale));
      left: calc(0.05cm * var(--scale));
      width: calc(0.7cm * var(--scale));
      height: calc(6cm * var(--scale));
      font-size: calc(11pt * var(--scale));
      align-items: center;
      justify-content: center;
      color: #fff;
      font-family: 'Nanum Gothic', 'Times New Roman', serif;
      letter-spacing: calc(0.03em * var(--scale));

      margin: 0;
      writing-mode: vertical-lr;
      text-orientation: sideways;
      transform: rotate(180deg);
      // dark outline with layers of box-shadow
      text-shadow:
        0 0 1px #000,
        0 0 2px #000,
        0 0 3px #000,
        0 0 4px #000,
        0 0 5px #000,
        0 0 6px #000,
        0 0 7px #000,
        0 0 8px #000;
    }
    .abilities {
      position: absolute;
      display: flex;
      flex-direction: column;
      margin: 0;
      bottom: calc(0.3cm * var(--scale));
      right: calc(0.15cm * var(--scale));
      left: calc(1cm * var(--scale));
      color: white;
      gap: 2px;
      & > li {
        margin-bottom: 0;
      }
      div.ability {
        display: flex;
        flex-direction: column;
        align-items: flex-start;
        justify-content: flex-start;
        align-content: flex-start;
        border-top: calc(0.25pt * var(--scale)) solid rgba(245, 235, 255, 0.42);
        border-left: none;
        border-right: none;
        border-bottom: none;
        box-shadow: inset 0 calc(1px * var(--scale)) calc(3px * var(--scale)) rgba(255,255,255,0.08);

        padding: calc(0.03cm * var(--scale)) calc(1pt * var(--scale)) 2px;
        background: linear-gradient(to bottom, #0000005e 80%, #0000 100%);
        color: #fff;
        font-family: 'Nanum Gothic', 'Times New Roman', serif;
        font-weight: 500;
        text-shadow:
          0 0 1px #000,
          0 0 2px #000,
          0 0 3px #000,
          0 0 4px #000,
          0 0 5px #000,
          0 0 6px #000,
          0 0 7px #000,
          0 0 8px #000;
        margin: 0;
        font-size: calc(7.5pt * var(--scale));
      }
    }
    .card-edition {
      position: absolute;
      right: calc(0.2cm * var(--scale));
      bottom: calc(0.04cm * var(--scale));
      display: flex;
      align-items: center;
      justify-content: flex-end;
      gap: calc(0.08cm * var(--scale));
      z-index: 6;
      color: rgba(210, 184, 96, 0.95);
      font-family: 'Nanum Gothic', 'Times New Roman', serif;
      font-size: calc(4.5pt * var(--scale));
      line-height: 1;
      font-weight: 700;
      letter-spacing: calc(0.015em * var(--scale));
      text-transform: uppercase;
      text-shadow:
        0 0 1px #000,
        0 0 2px #000,
        0 0 3px #000,
        0 0 4px #000;
    }
    .edition-value {
      color: rgba(210, 184, 96, 0.95);
      font-weight: 700;
      font-size: inherit;
      font-style: normal;
      letter-spacing: calc(0.015em * var(--scale));
      text-transform: uppercase;
      opacity: 0.92;
    }
    .ability-icons {
      position: relative;
      z-index: 3;
      // margin-bottom: calc(-0.35cm * var(--scale));
      // min-height: calc(0.8cm * var(--scale));
      padding-bottom: 10px;
    }
    .goodwillRank {
      display: flex;
      gap: 0cm;
      margin-bottom: calc(-0.35cm * var(--scale));
      margin-left: calc(0cm * var(--scale));
      li {
        list-style: none;
        margin-left: calc(0.05cm * var(--scale));
        width: 14px;
      }
      img {
        width: calc(0.75cm * var(--scale));
      }
    }
    .perLoop {
      float: right;
      display: flex;
      gap: calc(0cm * var(--scale));
      margin-right: calc(0.05cm * var(--scale));
      margin-bottom: calc(-0.35cm * var(--scale));
      li {
        list-style: none;
        margin-left: calc(-0.25cm * var(--scale));
      }
      .perLoopText {
        color: #fdfdfd;
        font-family: 'Nanum Gothic', 'Times New Roman', serif;
        font-weight: 400;
        font-size: calc(10pt * var(--scale));
        letter-spacing: calc(0.01em * var(--scale));
        text-shadow:
          0 0 1px #000,
          0 0 2px rgba(0, 0, 0, 0.8);
      }
      .perLoopInfinity {
        font-size: calc(11pt * var(--scale));
        line-height: 1;
        vertical-align: baseline;
        display: inline-block;
      }
    }
    .paranoia {
      position: absolute;
      display: flex;
      top: calc(0.18cm * var(--scale));
      left: calc(0.6cm * var(--scale));
      width: auto;
      height: auto;
      gap: calc(0.04cm * var(--scale));
      list-style: none;
      margin: 0;
      padding: 0;
      z-index: 2;
      align-items: flex-start;
      justify-content: flex-start;

      li {
        display: flex;
        align-items: center;
        justify-content: center;
      }

      img {
        width: 15px;
        height: 15px;
        object-fit: contain;
        filter: drop-shadow(0 0 calc(2px * var(--scale)) rgba(0, 0, 0, 0.32));
      }
    }
    .paranoia-limit-bar {
      position: absolute;
      top: calc(0.62cm * var(--scale) + 2px);
      left: 0;
      width: 90px;
      height: calc(0.25cm * var(--scale));
      display: flex;
      align-items: center;
      justify-content: center;
      background: rgba(0, 0, 0, 0.8);
      border-top-right-radius: calc(0.12cm * var(--scale));
      border-bottom-right-radius: calc(0.12cm * var(--scale));
      border-top-left-radius: 0;
      border-bottom-left-radius: 0;
      box-shadow:
        inset 0 0 calc(1px * var(--scale)) rgba(255, 255, 255, 0.08),
        0 0 calc(5px * var(--scale)) rgba(0, 0, 0, 0.45);
      z-index: 2;
      margin-top: calc(2px * var(--scale));
      overflow: hidden;

      &::before {
        content: '';
        position: absolute;
        inset: -1px;
        border-radius: inherit;
        background: transparent;
        box-shadow: 0 0 calc(6px * var(--scale)) rgba(255,255,255,0.18), 0 0 calc(10px * var(--scale)) rgba(0,0,0,0.7);
        filter: blur(calc(2px * var(--scale)));
        pointer-events: none;
      }

      span {
        color: rgba(242, 231, 202, 0.9);
        font-size: calc(6pt * var(--scale));
        line-height: 1;
        font-weight: 400;
        letter-spacing: calc(0.015em * var(--scale));
        text-transform: lowercase;
        font-family: 'Times New Roman', Times, serif;
      }
    }
    .paranoia-limit-number {
      position: absolute;
      top: calc(0.25cm * var(--scale));
      left: calc(0.15cm * var(--scale));
      height: calc(0.20cm * var(--scale));
      display: flex;
      align-items: center;
      justify-content: center;
      color: rgb(117 12 166);
      background: transparent;
      font-family: 'Lucida Handwriting', 'Brush Script MT', 'URW Chancery L', cursive;
      font-size: calc(18pt * var(--scale));
      font-weight: 700;
      font-style: italic;
      line-height: 1;
      z-index: 3;
      margin-right: calc(0.20cm * var(--scale));
      text-shadow:
        0 0 calc(2px * var(--scale)) rgba(165, 12, 124, 0.95),
        0 0 calc(5px * var(--scale)) rgba(205, 180, 255, 0.95),
        0 0 calc(7px * var(--scale)) rgba(205, 180, 255, 0.95),
        0 0 calc(10px * var(--scale)) rgba(165, 12, 124, 1);
      filter: drop-shadow(0 0 calc(2px * var(--scale)) rgba(201, 175, 255, 0.8));
      box-shadow: none;

      span {
        display: block;
      }
    }
    .tags {
      list-style: none;

      position: absolute;
      display: flex;
      flex-direction: column;
      align-items: flex-end;
      gap: calc(0.08cm * var(--scale));
      top: calc(1.75cm * var(--scale) + 2px);
      right: calc(0.05cm * var(--scale));
      height: calc(6cm * var(--scale));
      font-size: calc(9pt * var(--scale));
      font-family: 'Nanum Gothic', 'Times New Roman', serif;

      li {
        display: flex;
        align-items: center;
        justify-content: flex-end;
        width: 100%;
        margin: 0;

        div {
          display: inline-flex;
          align-items: center;
          justify-content: flex-end;
          box-sizing: border-box;
          padding: calc(0.015cm * var(--scale)) calc(0.2cm * var(--scale));
          border-top: calc(0.02cm * var(--scale)) solid rgba(32, 53, 49, 0.7);
          border-bottom: calc(0.02cm * var(--scale)) solid rgba(32, 53, 49, 0.7);
          border-left: calc(0.02cm * var(--scale)) solid rgba(32, 53, 49, 0.55);
          border-right: calc(0.02cm * var(--scale)) solid rgba(32, 53, 49, 0.55);
          border-radius: calc(0.18cm * var(--scale));
          margin: 0;
          font-size: calc(7pt * var(--scale));
          color: rgba(248, 240, 212, 0.94);
          letter-spacing: calc(0.02em * var(--scale));
          text-shadow:
            0 0 calc(1px * var(--scale)) rgba(255, 244, 208, 0.3),
            calc(1px * var(--scale)) calc(1px * var(--scale)) calc(2px * var(--scale)) rgba(0, 0, 0, 0.8);
          background: linear-gradient(
            to right,
            rgba(31, 63, 60, 0.72),
            rgba(56, 100, 94, 0.58),
            rgba(31, 63, 60, 0.72)
          );
          box-shadow:
            0 0 calc(8px * var(--scale)) rgba(0, 0, 0, 0.15),
            inset 0 0 calc(2px * var(--scale)) rgba(255, 255, 255, 0.06),
            inset 0 0 calc(12px * var(--scale)) rgba(179, 214, 200, 0.04);
          width: calc(66px * var(--scale));
          max-width: calc(66px * var(--scale));
          height: calc(0.34cm * var(--scale));
          line-height: 1.1;
          padding: calc(0.01cm * var(--scale)) calc(0.2cm * var(--scale));
          text-align: end;
          filter: blur(0.05px) drop-shadow(0 0 calc(1px * var(--scale)) rgba(195, 230, 207, 0.06));
        }
      }
    }
  }
  li {
    list-style: none;
  }
  ul {
    padding: 0;
  }
</style>
