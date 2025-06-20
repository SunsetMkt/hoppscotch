<template>
  <div>
    <header
      ref="headerRef"
      data-tauri-drag-region
      class="grid grid-cols-5 grid-rows-1 gap-2 overflow-x-auto overflow-y-hidden p-2"
    >
      <div
        data-tauri-drag-region
        class="col-span-2 flex items-center justify-between space-x-2"
        :style="{
          paddingTop: platform.ui?.appHeader?.paddingTop?.value,
          paddingLeft: platform.ui?.appHeader?.paddingLeft?.value,
        }"
      >
        <div class="flex">
          <tippy
            v-if="kernelMode === 'desktop'"
            interactive
            trigger="click"
            theme="popover"
            :on-shown="() => instanceSwitcherRef.focus()"
          >
            <div class="flex items-center cursor-pointer">
              <div class="flex">
                <span
                  class="!font-bold uppercase tracking-wide !text-secondaryDark pr-1"
                >
                  {{ instanceDisplayName }}
                </span>
                <span
                  v-if="
                    currentState.status === 'connected' &&
                    'type' in currentState.instance &&
                    currentState.instance.type === 'vendored'
                  "
                  class="!font-bold uppercase tracking-wide !text-secondaryDark pr-1"
                >
                  {{ platform.instance.displayConfig.description }}
                </span>
              </div>
              <IconChevronDown class="h-4 w-4 text-secondaryDark" />
            </div>
            <template #content="{ hide }">
              <div
                ref="instanceSwitcherRef"
                class="flex flex-col focus:outline-none min-w-64"
                tabindex="0"
                @keyup.escape="hide()"
              >
                <InstanceSwitcher @close-dropdown="hide()" />
              </div>
            </template>
          </tippy>
          <HoppButtonSecondary
            v-else
            class="!font-bold uppercase tracking-wide !text-secondaryDark hover:bg-primaryDark focus-visible:bg-primaryDark"
            :label="t('app.name')"
            to="/"
          />
        </div>
      </div>
      <div
        data-tauri-drag-region
        class="col-span-1 flex items-center justify-between space-x-2"
      >
        <AppSpotlightSearch />
      </div>
      <div
        data-tauri-drag-region
        class="col-span-2 flex items-center justify-between space-x-2"
      >
        <div class="flex">
          <tippy
            v-if="
              kernelMode === 'web' &&
              downloadableLinks &&
              downloadableLinks.length > 0
            "
            interactive
            trigger="click"
            theme="popover"
            :on-shown="() => downloadableLinksRef.focus()"
          >
            <HoppButtonSecondary
              :icon="IconDownload"
              class="rounded hover:bg-primaryDark focus-visible:bg-primaryDark"
            />
            <template #content="{ hide }">
              <div
                ref="downloadableLinksRef"
                class="flex flex-col focus:outline-none"
                tabindex="0"
                @keyup.escape="hide()"
              >
                <template v-for="link in downloadableLinks" :key="link.id">
                  <HoppButtonSecondary
                    v-if="link.show ?? true"
                    :icon="link.icon"
                    :label="link.text(t)"
                    :blank="true"
                    class="rounded hover:bg-primaryDark focus-visible:bg-primaryDark justify-between"
                    :to="
                      link.action.type === 'link' ? link.action.href : undefined
                    "
                    @click="
                      link.action.type === 'custom' ? link.action.do() : null
                    "
                  />
                </template>
              </div>
            </template>
          </tippy>

          <HoppButtonSecondary
            v-tippy="{ theme: 'tooltip', allowHTML: true }"
            :title="`${
              mdAndLarger ? t('support.title') : t('app.options')
            } <kbd>?</kbd>`"
            :icon="IconLifeBuoy"
            class="rounded hover:bg-primaryDark focus-visible:bg-primaryDark"
            @click="invokeAction('modals.support.toggle')"
          />
        </div>
        <div
          class="flex"
          :class="{
            'flex-row-reverse gap-2':
              workspaceSelectorFlagEnabled && !currentUser,
          }"
        >
          <div
            v-if="currentUser === null"
            class="inline-flex items-center space-x-2"
          >
            <HoppButtonSecondary
              v-if="!workspaceSelectorFlagEnabled"
              :icon="IconUploadCloud"
              :label="t('header.save_workspace')"
              class="!focus-visible:text-emerald-600 !hover:text-emerald-600 hidden h-8 border border-emerald-600/25 bg-emerald-500/10 !text-emerald-500 hover:border-emerald-600/20 hover:bg-emerald-600/20 focus-visible:border-emerald-600/20 focus-visible:bg-emerald-600/20 md:flex"
              @click="invokeAction('modals.login.toggle')"
            />
            <HoppButtonPrimary
              :label="t('header.login')"
              class="h-8"
              @click="invokeAction('modals.login.toggle')"
            />
          </div>
          <TeamsMemberStack
            v-else-if="
              currentUser !== null &&
              workspace.type === 'team' &&
              selectedTeam &&
              selectedTeam.teamMembers.length > 1
            "
            :team-members="selectedTeam.teamMembers"
            show-count
            class="mx-2"
            @handle-click="handleTeamEdit()"
          />
          <div
            v-if="workspaceSelectorFlagEnabled || currentUser"
            class="inline-flex items-center space-x-2"
          >
            <div
              class="flex h-8 divide-x divide-emerald-600/25 rounded border border-emerald-600/25 bg-emerald-500/10 focus-within:divide-emerald-600/20 focus-within:border-emerald-600/20 focus-within:bg-emerald-600/20 hover:divide-emerald-600/20 hover:border-emerald-600/20 hover:bg-emerald-600/20"
            >
              <HoppButtonSecondary
                v-tippy="{ theme: 'tooltip' }"
                :title="t('team.invite_tooltip')"
                :icon="IconUserPlus"
                class="!focus-visible:text-emerald-600 !hover:text-emerald-600 !text-emerald-500"
                @click="handleInvite()"
              />
              <HoppButtonSecondary
                v-if="
                  currentUser &&
                  workspace.type === 'team' &&
                  selectedTeam &&
                  selectedTeam?.myRole === 'OWNER'
                "
                v-tippy="{ theme: 'tooltip' }"
                :title="t('team.edit')"
                :icon="IconSettings"
                class="!focus-visible:text-emerald-600 !hover:text-emerald-600 !text-emerald-500"
                @click="handleTeamEdit()"
              />
            </div>
            <tippy
              interactive
              trigger="click"
              theme="popover"
              :on-shown="() => accountActions.focus()"
            >
              <HoppSmartSelectWrapper
                class="!text-blue-500 !focus-visible:text-blue-600 !hover:text-blue-600"
              >
                <HoppButtonSecondary
                  v-tippy="{ theme: 'tooltip' }"
                  :title="t('workspace.change')"
                  :label="mdAndLarger ? workspaceName : ``"
                  :icon="workspace.type === 'personal' ? IconUser : IconUsers"
                  class="!focus-visible:text-blue-600 !hover:text-blue-600 h-8 rounded border border-blue-600/25 bg-blue-500/10 pr-8 !text-blue-500 hover:border-blue-600/20 hover:bg-blue-600/20 focus-visible:border-blue-600/20 focus-visible:bg-blue-600/20"
                />
              </HoppSmartSelectWrapper>
              <template #content="{ hide }">
                <div
                  ref="accountActions"
                  class="flex flex-col focus:outline-none"
                  tabindex="0"
                  @keyup.escape="hide()"
                  @click="hide()"
                >
                  <WorkspaceSelector />
                </div>
              </template>
            </tippy>
            <span v-if="currentUser" class="px-2">
              <tippy
                interactive
                trigger="click"
                theme="popover"
                :on-shown="() => tippyActions.focus()"
              >
                <HoppSmartPicture
                  v-tippy="{
                    theme: 'tooltip',
                  }"
                  :name="currentUser.uid"
                  :title="
                    currentUser.displayName ||
                    currentUser.email ||
                    t('profile.default_hopp_displayname')
                  "
                  indicator
                  :indicator-styles="
                    network.isOnline ? 'bg-green-500' : 'bg-red-500'
                  "
                />
                <template #content="{ hide }">
                  <div
                    ref="tippyActions"
                    class="flex flex-col focus:outline-none"
                    tabindex="0"
                    @keyup.p="profile.$el.click()"
                    @keyup.s="settings.$el.click()"
                    @keyup.d="dashboard.$el.click()"
                    @keyup.l="logout.$el.click()"
                    @keyup.escape="hide()"
                  >
                    <div class="flex flex-col px-2">
                      <span class="inline-flex truncate font-semibold">
                        {{
                          currentUser.displayName ||
                          t("profile.default_hopp_displayname")
                        }}
                      </span>
                      <span
                        class="inline-flex truncate text-secondaryLight text-tiny"
                        >{{ currentUser.email }}</span
                      >
                    </div>
                    <hr />
                    <HoppSmartItem
                      ref="profile"
                      to="/profile"
                      :icon="IconUser"
                      :label="t('navigation.profile')"
                      :shortcut="['P']"
                      @click="hide()"
                    />
                    <HoppSmartItem
                      ref="settings"
                      to="/settings"
                      :icon="IconSettings"
                      :label="t('navigation.settings')"
                      :shortcut="['S']"
                      @click="hide()"
                    />
                    <HoppSmartItem
                      v-if="isUserAdmin"
                      ref="dashboard"
                      to="/admin/dashboard"
                      :icon="IconLayoutDashboard"
                      :label="t('navigation.admin_dashboard')"
                      :shortcut="['D']"
                      @click="hide()"
                    />
                    <FirebaseLogout
                      ref="logout"
                      :shortcut="['L']"
                      @confirm-logout="hide()"
                    />
                  </div>
                </template>
              </tippy>
            </span>
          </div>
        </div>
      </div>
    </header>
    <AppBanner
      v-if="bannerContent"
      :banner="bannerContent"
      @dismiss="dismissBanner"
    />
    <TeamsModal :show="showTeamsModal" @hide-modal="showTeamsModal = false" />

    <template v-if="workspace.type === 'team' && workspace.teamID">
      <component
        :is="platform.ui.additionalTeamInviteComponent"
        v-if="
          platform.ui?.additionalTeamInviteComponent &&
          workspace.type === 'team' &&
          workspace.teamID
        "
        :show="showModalInvite"
        :editing-team-i-d="editingTeamID"
        @hide-modal="displayModalInvite(false)"
      />

      <TeamsInvite
        v-else
        :show="showModalInvite"
        :editing-team-i-d="editingTeamID"
        @hide-modal="displayModalInvite(false)"
      />
    </template>

    <component
      :is="platform.ui.additionalTeamEditComponent"
      v-if="platform.ui?.additionalTeamEditComponent"
      :show="showModalEdit"
      :editing-team="editingTeamName"
      :editing-team-i-d="editingTeamID"
      @hide-modal="displayModalEdit(false)"
      @invite-team="inviteTeam(editingTeamName, editingTeamID)"
      @refetch-teams="refetchTeams"
    />

    <TeamsEdit
      v-else
      :show="showModalEdit"
      :editing-team="editingTeamName"
      :editing-team-i-d="editingTeamID"
      @hide-modal="displayModalEdit(false)"
      @invite-team="inviteTeam(editingTeamName, editingTeamID)"
      @refetch-teams="refetchTeams"
    />

    <HoppSmartConfirmModal
      :show="confirmRemove"
      :title="t('confirm.remove_team')"
      @hide-modal="confirmRemove = false"
      @resolve="deleteTeam"
    />
  </div>
</template>

<script setup lang="ts">
import { getKernelMode } from "@hoppscotch/kernel"

import { useI18n } from "@composables/i18n"
import { useReadonlyStream } from "@composables/stream"
import { defineActionHandler, invokeAction } from "@helpers/actions"
import { breakpointsTailwind, useBreakpoints, useNetwork } from "@vueuse/core"
import { useService } from "dioc/vue"
import * as TE from "fp-ts/TaskEither"
import { pipe } from "fp-ts/function"
import { computed, onMounted, reactive, ref, watch } from "vue"
import { useToast } from "~/composables/toast"
import { GetMyTeamsQuery, TeamAccessRole } from "~/helpers/backend/graphql"
import { deleteTeam as backendDeleteTeam } from "~/helpers/backend/mutations/Team"
import { platform } from "~/platform"
import {
  BANNER_PRIORITY_LOW,
  BannerContent,
  BannerService,
} from "~/services/banner.service"
import { WorkspaceService } from "~/services/workspace.service"
import { InstanceSwitcherService } from "~/services/instance-switcher.service"
import IconDownload from "~icons/lucide/download"
import IconLifeBuoy from "~icons/lucide/life-buoy"
import IconSettings from "~icons/lucide/settings"
import IconUploadCloud from "~icons/lucide/upload-cloud"
import IconUser from "~icons/lucide/user"
import IconUserPlus from "~icons/lucide/user-plus"
import IconUsers from "~icons/lucide/users"
import IconChevronDown from "~icons/lucide/chevron-down"
import IconLayoutDashboard from "~icons/lucide/layout-dashboard"
import { AdditionalLinksService } from "~/services/additionalLinks.service"

const t = useI18n()
const toast = useToast()
const kernelMode = getKernelMode()
const instanceSwitcherService =
  kernelMode === "desktop" ? useService(InstanceSwitcherService) : null
const instanceSwitcherRef =
  kernelMode === "desktop" ? ref<any | null>(null) : ref(null)
const downloadableLinksRef =
  kernelMode === "web" ? ref<any | null>(null) : ref(null)

const isUserAdmin = ref(false)

const currentState =
  kernelMode === "desktop" && instanceSwitcherService
    ? useReadonlyStream(
        instanceSwitcherService.getStateStream(),
        instanceSwitcherService.getCurrentState().value
      )
    : ref({
        status: "disconnected",
        instance: { displayName: "Hoppscotch" },
      })

const instanceDisplayName = computed(() => {
  if (currentState.value.status !== "connected") {
    return "Hoppscotch"
  }
  return currentState.value.instance.displayName
})

/**
 * Feature flag to enable the workspace selector login conversion
 */
const workspaceSelectorFlagEnabled = computed(
  () => !!platform.platformFeatureFlags.workspaceSwitcherLogin?.value
)

/**
 * Show the dashboard link if the user is not on the default cloud instance and is an admin
 */
onMounted(async () => {
  const { organization } = platform

  if (!organization || organization.isDefaultCloudInstance) return

  const orgInfo = await organization.getOrgInfo()

  if (orgInfo) {
    isUserAdmin.value = !!orgInfo.isAdmin
  }
})

const showTeamsModal = ref(false)

const breakpoints = useBreakpoints(breakpointsTailwind)
const mdAndLarger = breakpoints.greater("md")

const banner = useService(BannerService)
const bannerContent = computed(() => banner.content.value?.content)
let offlineBannerID: number | null = null

const offlineBanner: BannerContent = {
  type: "warning",
  text: (t) => t("helpers.offline"),
  alternateText: (t) => t("helpers.offline_short"),
  score: BANNER_PRIORITY_LOW,
  dismissible: true,
}

const additionalLinks = useService(AdditionalLinksService)

platform.additionalLinks?.forEach((linkSet) => {
  useService(linkSet)
})

const downloadableLinks = computed(() => {
  if (kernelMode !== "web") return null

  const headerDownloadableLink = additionalLinks?.getLinkSet(
    "HEADER_DOWNLOADABLE_LINKS"
  )

  if (!headerDownloadableLink) return null

  return headerDownloadableLink.getLinks().value
})

// Show the offline banner if the app is offline
const network = reactive(useNetwork())
const isOnline = computed(() => network.isOnline)

watch(isOnline, () => {
  if (!isOnline.value) {
    offlineBannerID = banner.showBanner(offlineBanner)
    return
  }
  if (banner.content && offlineBannerID) {
    banner.removeBanner(offlineBannerID)
  }
})

const dismissBanner = () => {
  if (banner.content.value) {
    banner.removeBanner(banner.content.value.id)
  } else if (offlineBannerID) {
    banner.removeBanner(offlineBannerID)
    offlineBannerID = null
  }
}

const currentUser = useReadonlyStream(
  platform.auth.getProbableUserStream(),
  platform.auth.getProbableUser()
)

const confirmRemove = ref(false)
const teamID = ref<string | null>(null)

const selectedTeam = ref<GetMyTeamsQuery["myTeams"][number] | undefined>()

// TeamList-Adapter
const workspaceService = useService(WorkspaceService)
const teamListAdapter = workspaceService.acquireTeamListAdapter(null)
const myTeams = useReadonlyStream(teamListAdapter.teamList$, null)

const workspace = workspaceService.currentWorkspace

const workspaceName = computed(() => {
  return workspace.value.type === "personal"
    ? t("workspace.personal")
    : workspace.value.teamName
})

const refetchTeams = () => {
  teamListAdapter.fetchList()
}

watch(
  () => myTeams.value,
  (newTeams) => {
    const space = workspace.value

    if (newTeams && space.type === "team" && space.teamID) {
      const team = newTeams.find((team) => team.id === space.teamID)
      if (team) {
        selectedTeam.value = team
        // Update the workspace name if it's not the same as the updated team name
        if (team.name !== space.teamName) {
          workspaceService.updateWorkspaceTeamName(team.name)
        }
      }
    }
  }
)

watch(
  () => workspace.value,
  (newWorkspace) => {
    if (newWorkspace.type === "team") {
      const team = myTeams.value?.find((t) => t.id === newWorkspace.teamID)
      if (team) {
        selectedTeam.value = team
      }
    }
  }
)

const showModalInvite = ref(false)
const showModalEdit = ref(false)

const editingTeamName = ref<{ name: string }>({ name: "" })
const editingTeamID = ref("")

const displayModalInvite = (show: boolean) => {
  showModalInvite.value = show
}

const displayModalEdit = (show: boolean) => {
  showModalEdit.value = show
  teamListAdapter.fetchList()
}

const inviteTeam = (team: { name: string }, teamID: string) => {
  editingTeamName.value = team
  editingTeamID.value = teamID
  displayModalInvite(true)
}

// Show the workspace selected team invite modal if the user is an owner of the team else show the default invite modal
const handleInvite = () => {
  if (!currentUser.value) return invokeAction("modals.login.toggle")

  if (
    workspace.value.type === "team" &&
    workspace.value.teamID &&
    selectedTeam.value?.myRole === "OWNER"
  ) {
    editingTeamID.value = workspace.value.teamID
    displayModalInvite(true)
  } else {
    showTeamsModal.value = true
  }
}

// Show the workspace selected team edit modal if the user is an owner of the team
const handleTeamEdit = () => {
  if (
    workspace.value.type === "team" &&
    workspace.value.teamID &&
    selectedTeam.value?.myRole === "OWNER"
  ) {
    editingTeamID.value = workspace.value.teamID
    editingTeamName.value = { name: selectedTeam.value.name }
    displayModalEdit(true)
  } else {
    noPermission()
  }
}

const deleteTeam = () => {
  if (!teamID.value) return
  pipe(
    backendDeleteTeam(teamID.value),
    TE.match(
      (err) => {
        // TODO: Better errors ? We know the possible errors now
        toast.error(`${t("error.something_went_wrong")}`)
        console.error(err)
      },
      () => {
        invokeAction("workspace.switch.personal")
        toast.success(`${t("team.deleted")}`)
      }
    )
  )() // Tasks (and TEs) are lazy, so call the function returned
}

// Template refs
const tippyActions = ref<any | null>(null)
const profile = ref<any | null>(null)
const settings = ref<any | null>(null)
const dashboard = ref<any | null>(null)
const logout = ref<any | null>(null)
const accountActions = ref<any | null>(null)

defineActionHandler("modals.team.edit", handleTeamEdit)

defineActionHandler("modals.team.invite", () => {
  if (
    selectedTeam.value?.myRole === "OWNER" ||
    selectedTeam.value?.myRole === "EDITOR"
  ) {
    inviteTeam({ name: selectedTeam.value.name }, selectedTeam.value.id)
  } else {
    noPermission()
  }
})

defineActionHandler(
  "user.login",
  () => {
    invokeAction("modals.login.toggle")
  },
  computed(() => !currentUser.value)
)

defineActionHandler("modals.team.delete", ({ teamId }) => {
  if (selectedTeam.value?.myRole !== TeamAccessRole.Owner) return noPermission()
  teamID.value = teamId
  confirmRemove.value = true
})

const noPermission = () => {
  toast.error(`${t("profile.no_permission")}`)
}
</script>
