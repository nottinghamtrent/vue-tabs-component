<template>
    <div class="tabs-component">
        <ul role="tablist" class="tabs-component-tabs">
            <li
                v-for="(tab, i) in tabs"
                :key="i"
                :class="{ 'is-active': tab.isActive, 'is-disabled': tab.isDisabled }"
                class="tabs-component-tab"
                role="presentation"
                v-show="tab.isVisible"
            >
                <a v-html="tab.header"
                   :aria-controls="tab.hash"
                   :aria-selected="tab.isActive"
                   @click="selectTab(tab.hash, $event)"
                   :href="tab.hash"
                   class="tabs-component-tab-a"
                   role="tab"
                ></a>
            </li>
        </ul>
        <div class="tabs-component-navigation">
            <span><button class="previous" @click="previousTab(true)">Previous</button></span>
            <span><button class="next" @click="nextTab(true)">Next</button></span>
        </div>
        <div class="tabs-component-panels">
            <slot/>
        </div>
    </div>
</template>

<script>
    import expiringStorage from '../expiringStorage';

    export default {
        props: {
            cacheLifetime: {
                default: 5,
            },
            options: {
                type: Object,
                required: false,
                default: () => ({
                    useUrlFragment: true,
                }),
            },
        },

        data: () => ({
            tabs: [],
            activeTabHash: '',
        }),

        computed: {
            storageKey() {
                return `vue-tabs-component.cache.${window.location.host}${window.location.pathname}`;
            },

            activeTabIndex() {
                const tab = this.findTab(this.activeTabHash);
                return this.tabs.indexOf(tab);
            }
        },

        created() {
            this.tabs = this.$children;
        },

        mounted() {
            window.addEventListener('hashchange', () => this.selectTab(window.location.hash));

            if (this.findTab(window.location.hash)) {
                this.selectTab(window.location.hash);
                return;
            }

            const previousSelectedTabHash = expiringStorage.get(this.storageKey);

            if (this.findTab(previousSelectedTabHash)) {
                this.selectTab(previousSelectedTabHash);
                return;
            }

            if (this.tabs.length) {
                this.selectTab(this.tabs[0].hash);
            }
        },

        methods: {
            findTab(hash) {
                return this.tabs.find(tab => tab.hash === hash);
            },

            selectTab(selectedTabHash, event) {
                // See if we should store the hash in the url fragment.
                if (event && !this.options.useUrlFragment) {
                    event.preventDefault();
                }

                const selectedTab = this.findTab(selectedTabHash);

                if (! selectedTab) {
                    return;
                }

                if (selectedTab.isDisabled) {
                    return;
                }

                this.tabs.forEach(tab => {
                    tab.isActive = (tab.hash === selectedTab.hash);
                });

                this.$emit('changed', { tab: selectedTab });

                this.activeTabHash = selectedTab.hash;

                expiringStorage.set(this.storageKey, selectedTab.hash, this.cacheLifetime);
            },

            setTabVisible(hash, visible) {
                const tab = this.findTab(hash);

                if (! tab) {
                    return;
                }

                tab.isVisible = visible;

                if (tab.isActive) {
                    // If tab is active, set a different one as active.
                    tab.isActive = visible;

                    this.tabs.every((tab, index, array) => {
                        if (tab.isVisible) {
                            tab.isActive = true;

                            return false;
                        }

                        return true;
                    });
                }
            },

            nextTab(cycle = false) {
                const currentIndex = this.activeTabIndex;
                let newIndex = currentIndex;

                // Loop through each 'next' tab until we find one we can goto (i.e. not disabled)
                do {
                    newIndex++;
                    if (newIndex === this.tabs.length) {
                        newIndex = cycle ? 0 : currentIndex;
                    }        
                } while(!this.canMoveToSelectedIndex(newIndex));

                this.selectTabByIndex(newIndex);
            },

            previousTab(cycle = false) {
                const currentIndex = this.activeTabIndex;
                let newIndex = currentIndex;

                // Loop through each 'previous' tab until we find one we can goto (i.e. not disabled)
                do {
                    newIndex--;
                    if (newIndex < 0) {
                        newIndex = cycle ? this.tabs.length - 1 : currentIndex;
                    }        

                } while(!this.canMoveToSelectedIndex(newIndex));

                this.selectTabByIndex(newIndex);
            },            

            canMoveToSelectedIndex(index) {
                const tabHash = this.tabs[index].hash;
                const newTab = this.findTab(tabHash);

                // If the tab is disabled, we can't select it
                if(newTab.isDisabled) {
                    return false;
                }
                
                return true;
            },

            selectTabByIndex(index) {
                const tab = this.tabs[index];
                const href = tab.hash;

                this.selectTab(href, null);
            },          
        },
    };
</script>
