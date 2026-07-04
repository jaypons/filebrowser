<template>
  <div v-show="active" @click="closeHovers" class="overlay"></div>
  <nav :class="{ active }">
    <template v-if="isLoggedIn">
      <button @click="toAccountSettings" class="action">
        <i class="material-icons">person</i>
        <span>{{ user.username }}</span>
      </button>
      <button
        class="action"
        @click="toRoot"
        :aria-label="$t('sidebar.myFiles')"
        :title="$t('sidebar.myFiles')"
      >
        <i class="material-icons">folder</i>
        <span>{{ $t("sidebar.myFiles") }}</span>
      </button>

      <div v-if="user.perm.create">
        <button
          @click="showHover('newDir')"
          class="action"
          :aria-label="$t('sidebar.newFolder')"
          :title="$t('sidebar.newFolder')"
        >
          <i class="material-icons">create_new_folder</i>
          <span>{{ $t("sidebar.newFolder") }}</span>
        </button>

        <button
          @click="showHover('newFile')"
          class="action"
          :aria-label="$t('sidebar.newFile')"
          :title="$t('sidebar.newFile')"
        >
          <i class="material-icons">note_add</i>
          <span>{{ $t("sidebar.newFile") }}</span>
        </button>
      </div>

      <div v-if="user.perm.admin">
        <button
          class="action"
          @click="toGlobalSettings"
          :aria-label="$t('sidebar.settings')"
          :title="$t('sidebar.settings')"
        >
          <i class="material-icons">settings_applications</i>
          <span>{{ $t("sidebar.settings") }}</span>
        </button>
      </div>
      <button
        v-if="canLogout"
        @click="logout"
        class="action"
        id="logout"
        :aria-label="$t('sidebar.logout')"
        :title="$t('sidebar.logout')"
      >
        <i class="material-icons">exit_to_app</i>
        <span>{{ $t("sidebar.logout") }}</span>
      </button>
    </template>
    <template v-else>
      <router-link
        v-if="!hideLoginButton"
        class="action"
        to="/login"
        :aria-label="$t('sidebar.login')"
        :title="$t('sidebar.login')"
      >
        <i class="material-icons">exit_to_app</i>
        <span>{{ $t("sidebar.login") }}</span>
      </router-link>

      <router-link
        v-if="signup"
        class="action"
        to="/login"
        :aria-label="$t('sidebar.signup')"
        :title="$t('sidebar.signup')"
      >
        <i class="material-icons">person_add</i>
        <span>{{ $t("sidebar.signup") }}</span>
      </router-link>
    </template>

    <div
      class="credits"
      v-if="isFiles && !disableUsedPercentage"
      style="width: 90%; margin: 2em 2.5em 3em 2.5em"
    >
      <progress-bar :val="usage.usedPercentage" size="small"></progress-bar>
      <br />
      {{ $t("sidebar.diskUsed", { used: usage.used, total: usage.total }) }}
    </div>

    <p class="credits">
      <span>
        <a @click="showAbout = true" style="cursor: pointer; font-weight: bold;">MIC File Manager v2.0.0</a>
      </span>
      <span>
        <a @click="help">{{ $t("sidebar.help") }}</a>
      </span>
    </p>

    <div v-if="showAbout" class="custom-about-backdrop" @click.self="showAbout = false">
      <div class="custom-about-modal">
        <button class="close-btn" @click="showAbout = false">&times;</button>
        
        <div class="modal-logo-container">
          <img src="/img/logo.svg" alt="MIC File Manager Logo" />
        </div>

        <div class="modal-content-area">
          <h3>MIC File Manager</h3>
          <p class="app-version">Version 2.0.0 (Stable Internal Release)</p>
          
          <div class="modal-divider"></div>
          
          <p class="app-description">
            This is a secure cloud storage environment optimized for rapid asset distribution, secure system backups, and encrypted storage configurations.
          </p>

          <table class="app-info-table">
            <tr>
              <td><strong>Deployment:</strong></td>
              <td>Internal Server Node Cluster</td>
            </tr>
            <tr>
              <td><strong>Maintained By:</strong></td>
              <td>SysAdmin Core Team</td>
            </tr>
            <tr>
              <td><strong>Engine Base:</strong></td>
              <td>Modified Go Engine & Vue 3</td>
            </tr>
          </table>
        </div>
        </div>
    </div>
    </nav>
</template>

<script>
import { reactive } from "vue";
import { mapActions, mapState } from "pinia";
import { useAuthStore } from "@/stores/auth";
import { useFileStore } from "@/stores/file";
import { useLayoutStore } from "@/stores/layout";

import * as auth from "@/utils/auth";
import {
  version,
  signup,
  hideLoginButton,
  disableExternal,
  disableUsedPercentage,
  noAuth,
  logoutPage,
  loginPage,
} from "@/utils/constants";
import { files as api } from "@/api";
import ProgressBar from "@/components/ProgressBar.vue";
import prettyBytes from "pretty-bytes";

const USAGE_DEFAULT = { used: "0 B", total: "0 B", usedPercentage: 0 };

export default {
  name: "sidebar",
  setup() {
    const usage = reactive(USAGE_DEFAULT);
    return { usage, usageAbortController: new AbortController() };
  },
  components: {
    ProgressBar,
  },
  data() {
    return {
      showAbout: false, // Tracks whether the custom popup window is open or hidden
    };
  },
  inject: ["$showError"],
  computed: {
    ...mapState(useAuthStore, ["user", "isLoggedIn"]),
    ...mapState(useFileStore, ["isFiles", "reload"]),
    ...mapState(useLayoutStore, ["currentPromptName"]),
    active() {
      return this.currentPromptName === "sidebar";
    },
    signup: () => signup,
    hideLoginButton: () => hideLoginButton,
    version: () => version,
    disableExternal: () => disableExternal,
    disableUsedPercentage: () => disableUsedPercentage,
    canLogout: () => !noAuth && (loginPage || logoutPage !== "/login"),
  },
  methods: {
    ...mapActions(useLayoutStore, ["closeHovers", "showHover"]),
    abortOngoingFetchUsage() {
      this.usageAbortController.abort();
    },
    async fetchUsage() {
      const path = this.$route.path.endsWith("/")
        ? this.$route.path
        : this.$route.path + "/";
      let usageStats = USAGE_DEFAULT;
      if (this.disableUsedPercentage) {
        return Object.assign(this.usage, usageStats);
      }
      try {
        this.abortOngoingFetchUsage();
        this.usageAbortController = new AbortController();
        const usage = await api.usage(path, this.usageAbortController.signal);
        usageStats = {
          used: prettyBytes(usage.used, { binary: true }),
          total: prettyBytes(usage.total, { binary: true }),
          usedPercentage: Math.round((usage.used / usage.total) * 100),
        };
      } finally {
        return Object.assign(this.usage, usageStats);
      }
    },
    toRoot() {
      this.$router.push({ path: "/files" });
      this.closeHovers();
    },
    toAccountSettings() {
      this.$router.push({ path: "/settings/profile" });
      this.closeHovers();
    },
    toGlobalSettings() {
      this.$router.push({ path: "/settings/global" });
      this.closeHovers();
    },
    help() {
      this.showHover("help");
    },
    logout: auth.logout,
  },
  watch: {
    $route: {
      handler(to) {
        if (to.path.includes("/files")) {
          this.fetchUsage();
        }
      },
      immediate: true,
    },
  },
  unmounted() {
    this.abortOngoingFetchUsage();
  },
};
</script>

<style scoped>
/* ======================================================= */
/* MODAL POP-UP SYSTEM LAYOUT BACKGROUND STYLES             */
/* ======================================================= */
.custom-about-backdrop {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0, 0, 0, 0.75); /* Dims out the application behind it */
  backdrop-filter: blur(5px);     /* Smooth premium frosted-glass blur */
  z-index: 999999;                /* Overrides all layers */
  display: flex;
  justify-content: center;
  align-items: center;
}

.custom-about-modal {
  background: #1b1d2a;            /* Premium dark block background layout */
  border: 1px solid rgba(255, 255, 255, 0.1);
  padding: 35px 30px;
  border-radius: 12px;
  width: 400px;
  max-width: 90%;
  position: relative;
  box-shadow: 0 15px 40px rgba(0, 0, 0, 0.6);
  color: #ffffff;
  text-align: center;
  animation: modalFadeInEffect 0.2s ease-out;
}

.close-btn {
  position: absolute;
  top: 10px;
  right: 18px;
  background: none;
  border: none;
  color: rgba(255, 255, 255, 0.4);
  font-size: 28px;
  cursor: pointer;
  transition: color 0.2s;
}

.close-btn:hover {
  color: #ffffff;
}

.modal-logo-container img {
  height: 75px;
  width: auto;
  margin-bottom: 12px;
}

.modal-content-area h3 {
  margin: 5px 0;
  font-size: 1.6rem;
  font-weight: 600;
  color: #ffffff;
}

.app-version {
  font-size: 0.85rem;
  color: rgba(255, 255, 255, 0.4);
  margin-top: -2px;
}

.modal-divider {
  height: 1px;
  background: rgba(255, 255, 255, 0.15);
  margin: 15px 0;
}

.app-description {
  font-size: 0.9rem;
  line-height: 1.5;
  color: rgba(255, 255, 255, 0.8);
  margin-bottom: 22px;
  text-align: left;
}

.app-info-table {
  width: 100%;
  font-size: 0.85rem;
  text-align: left;
  border-collapse: collapse;
}

.app-info-table td {
  padding: 6px 4px;
  color: rgba(255, 255, 255, 0.7);
}

@keyframes modalFadeInEffect {
  from { opacity: 0; transform: scale(0.96); }
  to { opacity: 1; transform: scale(1); }
}
</style>