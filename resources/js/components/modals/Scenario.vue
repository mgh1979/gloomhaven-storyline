<template>
    <div>
        <modal ref="modal">
            <template v-slot:body v-if="scenario">
                <div id="scenario-title" class="pl-6 border-b border-white2-25"
                     :class="{'pb-2': scenario.regions, 'pb-4': !scenario.regions}">
                    <h2 class="mdc-dialog__title p-0 leading-none">
                        {{ scenario.isVisible() ? scenario.name : '#' + scenario.id }}
                        <span class="text-sm text-white2-50">{{ scenario.coordinates.name }}</span>
                        <button type="button" data-mdc-dialog-action="close"
                                class="mdc-button absolute right-0 top-0 mt-4">
                            <i class="material-icons">close</i>
                        </button>
                    </h2>
                    <span v-if="scenario.regions" class="text-sm text-white2-50 font-bold">{{ scenario.regions.pluck('name').implode(', ') }}</span>
                </div>

                <div class="mdc-dialog__content" id="scenario-content">
                    <div class="xs:flex w-full -ml-2 mb-2">
                        <radio v-if="scenario.is_side"
                               class="whitespace-no-wrap"
                               id="hidden" group="states" label="Not unlocked"
                               :key="'hidden-' + stateKey"
                               :checked="scenario.isHidden()"
                               @changed="stateChanged"
                        ></radio>
                        <div class="flex w-full">
                            <radio id="incomplete" group="states" :label="$t('general.incomplete')"
                                   :key="'incomplete-' + stateKey"
                                   :checked="scenario.isIncomplete()"
                                   :disabled="scenario.isBlocked() || scenario.isRequired()"
                                   @changed="stateChanged"
                            ></radio>
                            <radio id="complete" group="states" :label="$t('general.complete')"
                                   :key="'complete-' + stateKey"
                                   :checked="scenario.isComplete()"
                                   :disabled="scenario.isBlocked() || scenario.isRequired()"
                                   @changed="stateChanged"
                            ></radio>
                            <div v-if="scenario.isVisible()"
                                 class="hidden sm:block ml-auto w-20"
                                 :class="{'sm:block': scenario.is_side, 'xs:block': !scenario.is_side}">
                                <webp :src="scenario.image()"
                                      :animate="true"
                                      :alt="scenario.name"/>
                            </div>
                        </div>
                    </div>

                    <template v-if="scenario.isVisible()">

                        <div v-if="scenario.requirements" class="mb-2 flex items-center" style="margin-left: -2px;">
                            <i v-if="scenario.isRequired() || scenario.isBlocked()"
                               class="material-icons text-incomplete text-2xl mr-2">highlight_off</i>
                            <i v-else class="material-icons text-complete text-2xl mr-2">check_circle_outline</i>
                            Requirements: {{ scenario.requirements }}
                        </div>

                        <div class="mt-4 mb-2"
                             v-if="scenario.isVisible() && !scenario.isComplete() && !treasuresVisible">
                            <button class="mdc-button origin-left transform scale-75 mdc-button--raised"
                                    @click="treasuresVisible = true">
                                <i class="material-icons mdc-button__icon">attach_money</i>
                                <span class="mdc-button__label">Show treasures</span>
                            </button>
                        </div>

                        <div class="my-2"
                             v-if="(scenario.isComplete() || treasuresVisible) && scenario.treasures.isNotEmpty()">
                            <h2 class="text-white">Treasures</h2>
                            <div v-if="scenario.treasures.isNotEmpty()"
                                 v-for="(treasure, id) in scenario.treasures.items" :key="id"
                                 class="flex items-center -ml-2">
                                <checkbox
                                        :id="id"
                                        :label="'#' + id"
                                        :checked="scenario.isTreasureUnlocked(id)"
                                        @changed="treasureChanged"></checkbox>
                                <span v-if="scenario.isTreasureUnlocked(id)" class="ml-4">{{ treasure }}</span>
                            </div>
                        </div>
                        <p class="mb-2"
                           v-if="!scenario.isComplete() && scenario.treasures.isEmpty() && treasuresVisible">
                            No treasures available.
                        </p>

                        <template if="achievements" v-for="(x, is_global) in achievements">
                            <template v-for="(y, is_awarded) in x">
                                <div class="my-2 flex flex-col items-start"
                                     v-if="!y.isEmpty()">
                                    <h2 class="text-white">
                                        {{ is_awarded ? '' : 'Lost' }}
                                        {{ is_global ? 'Global' : 'Party' }}
                                        {{ y.count() > 1 ? 'Achievements' : 'Achievement' }}
                                    </h2>
                                    <div v-for="achievement in y"
                                         class="flex items-center">
                                        <button type="button"
                                                class="mdc-button normal-case -ml-2"
                                                @click="openAchievement(achievement.id)">
                                            <span class="mdc-button__label">{{ achievement.name }}</span>
                                        </button>
                                        <scenario-number class="ml-2"
                                                         v-for="scenario in requiredBy(achievement)"
                                                         :scenario="scenario" :key="scenario.id"/>
                                    </div>
                                </div>
                            </template>
                        </template>

                        <div class="my-2"
                             v-if="scenario.isComplete() && !scenario.rewards.isEmpty()">
                            <h2 class="text-white">
                                {{ scenario.rewards.count() > 1 ? 'Rewards' : 'Reward' }}
                            </h2>
                            <template v-if="typeof scenario.rewards.first() === 'string'">
                                <span>{{ scenario.rewards.join(', ') }}</span>
                            </template>
                            <template v-if="Array.isArray(scenario.rewards.first())"
                                      v-for="(rewards, index) in scenario.rewards">
                                <div>Conclusion <span class="uppercase">{{ n2l.convert(index) }}</span>:
                                    {{ rewards.join(', ') }}
                                </div>
                            </template>
                        </div>

                        <div class="mb-3 flex flex-col items-start">
                            <template v-for="(quest, index) in scenario.quests">
                                <button class="mdc-button normal-case -ml-2"
                                        @click="toggleQuest(index)">
                                    <span class="mdc-button__label font-title text-white">{{ $t(quest.name) }}</span>
                                    <i class="material-icons mdc-button__icon transform transition-transform duration-500 text-white"
                                       :class="{'rotate-0': questExpand[index], 'rotate-180': !questExpand[index]}">
                                        keyboard_arrow_up
                                    </i>
                                </button>
                                <transition-expand>
                                    <div v-if="questExpand[index]">
                                        <i18n :path="quest.description" tag="div">
                                            <template v-for="n in [1,2,3,4,5,6,7,8,9]" v-slot:[n]>
                                                <p class="mb-4">{{ $t('quest.' + quest.id + '.sections.' + n) }}</p>
                                            </template>
                                            <template v-slot:br>
                                                <br><br>
                                            </template>
                                        </i18n>
                                    </div>
                                </transition-expand>
                            </template>

                            <template v-if="scenario.hasCard()"
                                      v-for="(card, index) in scenario.cards">
                                <button class="mdc-button normal-case -ml-2"
                                        @click="toggleQuest(questCount + index)">
                                    <span class="mdc-button__label font-title text-white">{{ card.title }}</span>
                                    <i class="material-icons mdc-button__icon transform transition-transform duration-500 text-white"
                                       :class="{'rotate-0': questExpand[questCount + index], 'rotate-180': !questExpand[questCount + index]}">
                                        keyboard_arrow_up
                                    </i>
                                </button>
                                <transition-expand>
                                    <div v-if="questExpand[questCount + index]">
                                        <webp v-for="(image, index) in card.images"
                                              :key="card.id + '-' + index"
                                              :src="image"
                                              class="mb-4"
                                              :alt="card.title"/>
                                    </div>
                                </transition-expand>
                            </template>
                        </div>

                        <div class="mb-6 hidden">
                            <div class="mdc-text-field mdc-text-field--textarea w-full"
                                 ref="notes">
                                <textarea id="notes" @change="noteChanged" v-model="scenario.notes"
                                          class="mdc-text-field__input" rows="4" cols="40"></textarea>
                                <div class="mdc-notched-outline">
                                    <div class="mdc-notched-outline__leading"></div>
                                    <div class="mdc-notched-outline__notch">
                                        <label for="notes" class="mdc-floating-label">Notes</label>
                                    </div>
                                    <div class="mdc-notched-outline__trailing"></div>
                                </div>
                            </div>
                        </div>

                        <div class="flex items-center mb-6">
                            <button class="mdc-button mdc-button--raised" @click="openPages()">
                                <i class="material-icons mdc-button__icon">menu_book</i>
                                <span class="mdc-button__label">{{ $t('pages') }}</span>
                            </button>
                            <div class="ml-auto w-20"
                                 :class="{'sm:hidden': scenario.is_side, 'xs:hidden': !scenario.is_side}">
                                <webp :src="scenario.image()"
                                      :animate="true"
                                      :alt="scenario.name"/>
                            </div>
                        </div>
                    </template>
                </div>
                <footer class="mdc-dialog__actions flex justify-between px-5">
                    <div class="space-x-2">
                        <ScenarioNumber :scenario="scenario" v-for="scenario in prevScenarios" :key="scenario.id"/>
                    </div>
                    <div class="space-x-2">
                        <ScenarioNumber :scenario="scenario" v-for="scenario in nextScenarios" :key="scenario.id"/>
                    </div>
                </footer>
            </template>
        </modal>
        <pages v-if="scenario" ref="pages"
               :pages="scenario.pages"
        ></pages>
        <choose v-if="scenario && scenario.choices" ref="choose"
                :scenario-ids="scenario.choices"
                @scenario-chosen="scenarioChosen"
                @closing="chooseModalClosing"
        ></choose>
    </div>
</template>

<script>
    import ScenarioRepository from "../../repositories/ScenarioRepository";
    import AchievementRepository from "../../repositories/AchievementRepository";
    import {MDCTextField} from "@material/textfield/component";
    import {ScenarioState} from "../../models/ScenarioState";
    import PreloadImage from "../../services/PreloadImage";
    import N2l from "../../services/N2l";
    import ScenarioNumber from "../elements/ScenarioNumber";

    export default {
        components: {ScenarioNumber},
        data() {
            return {
                scenario: null,
                notes: null,
                stateKey: 1,
                questExpand: [],
                treasuresVisible: false,
                scenarioRepository: new ScenarioRepository(),
                achievementRepository: new AchievementRepository(),
                preloadImage: new PreloadImage(),
                n2l: new N2l()
            }
        },
        mounted() {
            this.$bus.$on('open-scenario', (data) => {
                this.open(data.id);
            });
        },
        computed: {
            prevScenarios() {
                return this.scenarioRepository.findMany(this.scenario.linked_from)
                    .where('state', ScenarioState.complete);
            },
            nextScenarios() {
                if (this.scenario.isComplete()) {
                    return this.scenarioRepository.findMany(this.scenario.links_to)
                        .where('state', '!=', ScenarioState.hidden);
                }

                return collect();
            },
            achievements() {
                if (this.scenario.isComplete()) {
                    let awarded = this.achievementRepository.findMany(this.scenario.achievements_awarded)
                        .where('_awarded', '=', true);
                    let lost = this.achievementRepository.findMany(this.scenario.achievements_lost)
                        .where('_awarded', '=', false);

                    return [[
                        lost.where('type', '=', 'party'),
                        awarded.where('type', '=', 'party')
                    ], [
                        lost.where('type', '=', 'global'),
                        awarded.where('type', '=', 'global')
                    ]];
                }

                return null;
            },
            questCount() {
                return this.scenario.quests.length || 0;
            }
        },
        methods: {
            stateChanged(state) {
                if (state === ScenarioState.complete && this.scenario.choices) {
                    this.$refs['choose'].open();
                } else {
                    this.scenarioRepository.changeState(this.scenario, state);
                }

                this.$bus.$emit('scenarios-updated');
            },
            noteChanged() {
                this.scenario.store();
            },
            treasureChanged(id, checked) {
                this.scenario.unlockTreasure(id, checked);

                if (this.scenarioRepository.unlockTreasureScenario(this.scenario, id)) {
                    this.$bus.$emit('scenarios-updated');
                }
            },
            scenarioChosen(choice) {
                this.scenarioRepository.choose(this.scenario, choice);

                this.$bus.$emit('scenarios-updated');
            },
            chooseModalClosing(action) {
                if (action !== 'chosen') {
                    this.rerenderStateSelection();
                }
            },
            toggleQuest(index) {
                this.$set(this.questExpand, index, !this.questExpand[index]);
            },
            openPages() {
                this.$refs['pages'].open();
            },
            rerenderStateSelection() {
                this.stateKey++;
            },
            requiredBy(achievement) {
                return this.scenarioRepository.requiredBy(achievement);
            },
            open(id) {
                this.scenario = this.scenarioRepository.find(id);
                this.treasuresVisible = false;

                let questCount = this.questCount + this.scenario.cards.count();
                this.questExpand = new Array(questCount);
                if (this.scenario.hasCard()) {
                    this.scenario.cards.each((card) => {
                        card.images.forEach((image) => {
                            this.preloadImage.handle(image);
                        });
                    });
                }

                this.rerenderStateSelection();

                this.$nextTick(() => {
                    this.$refs['modal'].open();
                    this.$nextTick(() => {
                        this.$refs['pages'].preload();
                        const notes = this.$refs['notes'];
                        if (notes) {
                            new MDCTextField(notes);
                        }
                    });
                });
            },
            close() {
                this.$refs['modal'].close();
            },
            openAchievement(id) {
                this.close();
                this.$bus.$emit('open-achievement', {
                    id: id
                });
            }
        }
    }
</script>

<style lang="scss">
    .mdc-notched-outline__notch {
        border-right: none;
        border-left: none;
    }
</style>

<i18n>
    {
    "en": {
    "pages": "pages"
    },
    "de": {
    "pages": "seiten"
    },
    "fr": {
    "pages": "pages"
    }
    }
</i18n>