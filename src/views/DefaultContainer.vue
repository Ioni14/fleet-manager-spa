<template>
    <div class="app" v-infinite-scroll="loadMore" :infinite-scroll-disabled="$store.state.infiniteScrollDisabled" infinite-scroll-distance="500">
        <AppHeader fixed>
            <SidebarToggler class="d-lg-none" display="md" mobile/>
            <router-link class="navbar-brand" to="/my-fleet">
                <img class="navbar-brand-full" src="@img/logo_fm_blue.svg" alt="SC Fleet Manager" height="40">
                <img class="navbar-brand-minimized" src="@img/icon_fm_blue.svg" alt="FM" height="40">
            </router-link>
            <SidebarToggler class="d-md-down-none" display="lg" :defaultOpen="true" ref="sidebarDesktop"/>
            <div v-if="displayEnv" class="p-3 bg-danger text-white text-uppercase">{{ environment }}</div>
            <b-navbar-nav class="ml-auto">
                <b-nav-item v-if="hasHeaderFeedbackForm" class="px-2" v-b-modal.modal-give-feedback><i class="fa fa-comment"></i><div class="px-2 d-none d-sm-inline-block"> Give feedback</div></b-nav-item>
                <router-link to="Profile">
                    <b-nav-text v-if="profile !== null" class="px-3 d-none d-sm-inline-block">Welcome, {{ profile.nickname }}</b-nav-text>
                </router-link>
                <b-nav-text v-if="profile !== null && profile.coins > 0" class="px-3 d-none d-sm-inline-block"><img src="@img/coin.svg" title="FM Coins" alt="FM Coins" height="30"> {{ profile.coins }}</b-nav-text>
                <b-nav-item v-if="$auth.isAuthenticated" class="px-3" @click="logout"><i class="fas fa-sign-out-alt"></i><div class="px-2 d-none d-sm-inline-block"> Logout</div></b-nav-item>
                <b-nav-item v-else class="px-3" @click="login"><i class="fas fa-sign-in-alt"></i> Login</b-nav-item>
            </b-navbar-nav>
        </AppHeader>
        <div class="app-body">
            <AppSidebar fixed>
                <SidebarNav :navItems="nav"></SidebarNav>
                <SidebarMinimizer/>
            </AppSidebar>
            <main class="main">
                <div class="container-fluid mt-4">
                    <router-view></router-view>
                </div>
            </main>
        </div>
        <TheFooter class="font-lg">
            <div>
                <router-link :to="{ name: 'Home' }">Fleet Manager</router-link>
                <span>{{ lastVersion }}</span>
                <span class="ml-1">&copy; 2018 - {{ actualYear }}</span>
                - <router-link to="/privacy-policy">Privacy policy</router-link>.
                Star Citizen is a product of Cloud Imperium Rights LLC and Cloud Imperium Rights Ltd.
            </div>
            <b-nav class="ml-auto">
                <b-nav-item link-classes="p-2" v-b-modal.modal-give-feedback><i class="fa fa-comment"></i><div class="px-2 d-none d-sm-inline-block"> Give feedback</div></b-nav-item>
                <b-nav-text class="p-2">–</b-nav-text>
                <b-nav-item link-classes="p-2" v-b-modal.modal-patch-notes>Patch notes</b-nav-item>
                <b-nav-text class="p-2">–</b-nav-text>
                <b-nav-item href="https://discord.gg/f6mrA3Y" target="_blank" link-classes="p-2"><i class="fab fa-discord" style="font-size: 1.4rem;"></i></b-nav-item>
                <b-nav-item href="https://github.com/Ioni14/fleet-manager-api" target="_blank" link-classes="p-2"><i class="fab fa-github" style="font-size: 1.4rem;"></i></b-nav-item>
            </b-nav>
        </TheFooter>

        <b-modal id="modal-patch-notes" ref="modalPatchNotes" size="lg" centered scrollable title="What's new?" hide-footer @show="onShowPatchNotes">
            <div v-for="patchNote in patchNotes" :key="patchNote.id">
                <h5>{{ patchNote.title }}</h5>
                <p class="small">{{ formatPatchNoteDate(patchNote.createdAt) }}</p>
                <p v-html="nl2br(patchNote.body)"></p>
                <p v-if="patchNote.link"><a :href="patchNote.link" target="_blank">{{ patchNote.link }}</a></p>
            </div>
        </b-modal>
        <b-modal id="modal-give-feedback" ref="modalGiveFeedback" size="lg" centered title="Give feedback" hide-footer>
            <CreateGiveFeedbackModal></CreateGiveFeedbackModal>
        </b-modal>
    </div>
</template>

<script>
    import axios from 'axios';
    import {
        Header as AppHeader,
        SidebarToggler,
        Sidebar as AppSidebar,
        SidebarMinimizer,
        SidebarNav,
        Footer as TheFooter
    } from '@coreui/vue';
    import { mapState } from 'vuex';
    import Config from '@config/config.json';
    import Version from '@/version.json';
    import CreateGiveFeedbackModal from '@/components/CreateGiveFeedbackModal';
    import moment from 'moment-timezone';
    import bus from '@/bus';

    export default {
        name: 'DefaultContainer',
        components: {
            AppHeader,
            AppSidebar,
            TheFooter,
            SidebarNav,
            SidebarToggler,
            SidebarMinimizer,
            CreateGiveFeedbackModal,
        },
        data() {
            return {
                patchNotes: [],
                versionCommitHash: null,
            };
        },
        created() {
            bus.$on('updateMyOrganizations', this.loadOrgaList);
            bus.$on('updatedProfile', this.updatedProfile);
            this.loadNewPatchNoteIfNew();
            this.loadVersionCommitHash();
            if (this.$store.state.myOrgasList === null) {
                this.loadOrgaList();
            }
        },
        computed: {
            ...mapState(['accessToken', 'profile']),
            hasHeaderFeedbackForm() {
                return Config.environment === 'beta';
            },
            lastVersion() {
                let version = Version.version;
                if (Config.environment !== 'prod') {
                    version += `-${Config.environment}${this.versionCommitHash !== null ? '-'+this.versionCommitHash : ''}`;
                }

                return version;
            },
            actualYear() {
                return (new Date()).getFullYear();
            },
            nav() {
                const nav = [];
                const myOrgasItems = [];
                if (this.$store.state.myOrgasList !== null) {
                    for (const orga of this.$store.state.myOrgasList) {
                        if (!orga.joined) {
                            continue;
                        }
                        myOrgasItems.push({
                            name: orga.name,
                            url: `/my-organizations/${orga.sid}`,
                            icon: ' ',
                            class: 'orga-item-menu',
                            attributes: {
                                style: 'background-image: url('+orga.logoUrl+');'
                            },
                        });
                    }
                }
                nav.push(
                    {
                        name: 'My Fleet',
                        url: `/my-fleet`,
                        icon: 'fas fa-space-shuttle',
                        attributes: {
                            disabled: !this.profile,
                        },
                    },
                    {
                        name: 'My organizations',
                        url: '/my-organizations',
                        icon: 'fas fa-users',
                        attributes: {
                            disabled: !this.profile,
                        },
                    },
                    ...myOrgasItems,
                    {
                        name: "Profile",
                        url: '/profile',
                        icon: 'fas fa-user',
                        class: 'mt-auto',
                        attributes: {
                            disabled: !this.profile,
                        },
                    },
                );

                return nav;
            },
            environment() {
                return Config.environment;
            },
            displayEnv() {
                return Config.environment === 'beta';
            }
        },
        methods: {
            login() {
                this.$auth.loginWithRedirect();
            },
            logout() {
                this.$auth.logout({
                    returnTo: window.location.origin
                });
            },
            formatPatchNoteDate(date) {
                return moment(date).format('LLL');
            },
            async giveFeedback() {
                if (!this.accessToken) {
                    return;
                }
                try {
                    await axios.get(`${Config.api_base_url}/api/support/give-feedback`, {
                        headers: {
                            Authorization: `Bearer ${this.accessToken}`,
                        }
                    });
                } catch (err) {
                    if (err.response && (err.response.status === 401 || err.response.status === 403)) {
                        this.$toastr.e('You have been disconnected. Please login again.');
                        this.$router.push({ name: 'Home' });
                        return;
                    }
                    const data = response.data;
                    for (const violation of data.violations.violations) {
                        switch (violation.propertyPath) {
                            case '':
                                this.globalViolation = violation.title;
                                break;
                            case 'model':
                                this.form.model.violation = violation.title;
                                break;
                            case 'pictureUrl':
                                this.form.imageUrl.violation = violation.title;
                                break;
                            case 'quantity':
                                this.form.quantity.violation = violation.title;
                                break;
                        }
                    }
                }
            },
            async loadVersionCommitHash() {
                if (Config.environment === 'prod') {
                    return;
                }
                try {
                    this.versionCommitHash = null;
                    const response = await axios.get('https://api.github.com/repos/Ioni14/fleet-manager-spa/branches/master');
                    if (response.data && response.data.commit) {
                        this.versionCommitHash = response.data.commit.sha.substring(0, 6);
                    }
                } catch (err) {
                    console.error(err);
                }
            },
            async loadNewPatchNoteIfNew() {
                if (!this.accessToken) {
                    return;
                }
                try {
                    const response = await axios.get(`${Config.api_base_url}/api/patch-note/has-new-patch-note`, {
                        headers: {
                            Authorization: `Bearer ${this.accessToken}`,
                        }
                    });
                    if (response.data.hasNewPatchNote) {
                        this.$bvModal.show('modal-patch-notes');
                    }
                } catch (err) {
                    if (err.response && (err.response.status === 401 || err.response.status === 403)) {
                        this.$toastr.e('You have been disconnected. Please login again.');
                        this.$router.push({ name: 'Home' });
                        return;
                    }
                    console.error(err);
                }
            },
            async onShowPatchNotes() {
                try {
                    const response = await axios.get(`${Config.api_base_url}/api/patch-note/last-patch-notes`, {
                        headers: {
                            Authorization: `Bearer ${this.$store.state.accessToken}`,
                        }
                    });
                    this.patchNotes = response.data.patchNotes;
                } catch (err) {
                    if (err.response && (err.response.status === 401 || err.response.status === 403)) {
                        this.$toastr.e('You have been disconnected. Please login again.');
                        this.$router.push({ name: 'Home' });
                        return;
                    }
                    if (err.response.data.errorMessage) {
                        this.$toastr.e(err.response.data.errorMessage);
                        return;
                    }
                    this.$toastr.e('Sorry, an unexpected error has occurred when requesting the last patch notes. Please try again later.');
                }
            },
            async loadOrgaList() {
                try {
                    const response = await axios.get(`${Config.api_base_url}/api/my-organizations`, {
                        headers: {
                            Authorization: `Bearer ${this.$store.state.accessToken}`,
                        },
                    });
                    this.$store.commit('myOrgasList', response.data.organizations);
                } catch (err) {
                    if (err.response && (err.response.status === 401 || err.response.status === 403)) {
                        this.$toastr.e('You have been disconnected. Please login again.');
                        this.$router.push({ name: 'Home' });
                        return;
                    }
                    this.$toastr.e('Sorry, we are unable to retrieve your organizations. Please, try again later.');
                } finally {
                    this.listOfOrgasLoaded = true;
                }
            },
            async updatedProfile() {
                this.$toastr.s('Your profile has been updated!');
                try {
                    const response = await axios.get(`${Config.api_base_url}/api/profile`, {
                        headers: {
                            Authorization: `Bearer ${this.$store.state.accessToken}`,
                        }
                    });
                    this.$store.commit('profile', response.data);
                } catch (err) {
                    if (err.response && (err.response.status === 401 || err.response.status === 403)) {
                        this.$toastr.e('You have been disconnected. Please login again.');
                        this.$router.push({ name: 'Home' });
                        return;
                    }
                    console.error(err);
                }
            },
            nl2br(str) {
                return str.replace(/(?:\r\n|\r|\n)/g, '<br />');
            },
            loadMore() {
                bus.$emit('infinite-scroll-load-more');
            },
        }
    };
</script>
