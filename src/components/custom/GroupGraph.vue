<template>
	<div class="fill-height">
		<v-expansion-panels v-model="openPanel">
			<v-expansion-panel>
				<v-expansion-panel-title>Options</v-expansion-panel-title>
				<v-expansion-panel-text>
					<v-row>
						<!-- Legend -->
						<v-col cols="12" sm="3">
							<v-list-subheader>Legend</v-list-subheader>
							<v-list density="compact">
								<v-list-item
									v-for="(item, i) in legends"
									:key="i"
								>
									<template #prepend>
										<v-icon :color="item.color">{{
											item.icon || 'circle'
										}}</v-icon>
									</template>
									<v-list-item-title
										:style="{ color: item.textColor }"
									>
										{{ item.text }}
									</v-list-item-title>
								</v-list-item>
							</v-list>
						</v-col>

						<!-- View Mode -->
						<v-col cols="12" sm="3">
							<v-list-subheader>View Mode</v-list-subheader>
							<v-btn-toggle
								v-model="viewMode"
								mandatory
								density="compact"
								color="primary"
							>
								<v-btn value="associations" size="small"
									>Devices</v-btn
								>
								<v-btn value="groups" size="small"
									>Groups</v-btn
								>
								<v-btn value="nodes" size="small"
									>Node→Groups</v-btn
								>
								<v-btn value="shared" size="small"
									>Shared</v-btn
								>
							</v-btn-toggle>

							<div class="mt-3">
								<v-list-subheader>Layout</v-list-subheader>
								<v-btn-toggle
									v-model="layoutMode"
									mandatory
									density="compact"
									color="secondary"
								>
									<v-btn value="force" size="small"
										>Force</v-btn
									>
									<v-btn value="hierarchical" size="small"
										>Hierarchy</v-btn
									>
								</v-btn-toggle>
							</div>
						</v-col>

						<!-- Filters -->
						<v-col cols="12" sm="3">
							<v-list-subheader>Filters</v-list-subheader>
							<v-checkbox
								v-model="showDeadOnly"
								label="Dead / Failed nodes only"
								density="compact"
								hide-details
							/>
							<v-checkbox
								v-model="showLifelineGroups"
								label="Show lifeline groups"
								density="compact"
								hide-details
							/>
							<v-checkbox
								v-model="showEmptyGroups"
								label="Show empty groups"
								density="compact"
								hide-details
							/>
						</v-col>

						<!-- Stats -->
						<v-col cols="12" sm="3">
							<v-list-subheader>Stats</v-list-subheader>
							<v-list density="compact">
								<v-list-item>
									Total groups
									<template #append
										><strong>{{
											stats.totalGroups
										}}</strong></template
									>
								</v-list-item>
								<v-list-item>
									Groups with dead nodes
									<template #append>
										<strong
											:style="{
												color:
													stats.groupsWithDead > 0
														? '#8b0000'
														: '',
											}"
										>
											{{ stats.groupsWithDead }}
										</strong>
									</template>
								</v-list-item>
								<v-list-item>
									Shared nodes (multi-group)
									<template #append
										><strong>{{
											stats.sharedNodes
										}}</strong></template
									>
								</v-list-item>
								<v-list-item>
									Total associations
									<template #append
										><strong>{{
											stats.totalAssociations
										}}</strong></template
									>
								</v-list-item>
								<v-list-item>
									Dead/Failed nodes
									<template #append>
										<strong
											:style="{
												color:
													stats.deadNodes > 0
														? '#8b0000'
														: '#2DCC70',
											}"
										>
											{{ stats.deadNodes }}
										</strong>
									</template>
								</v-list-item>
							</v-list>
						</v-col>
					</v-row>

					<v-row class="mt-0">
						<v-col class="d-flex align-center flex-wrap ga-2">
							<v-btn
								color="primary"
								variant="tonal"
								size="small"
								:loading="loading"
								@click="loadAllAssociations"
								prepend-icon="refresh"
							>
								Refresh
							</v-btn>
							<v-btn
								:color="demoMode ? 'warning' : 'secondary'"
								variant="tonal"
								size="small"
								@click="toggleDemoMode"
								prepend-icon="science"
							>
								{{ demoMode ? 'Exit Demo' : 'Demo Mode' }}
							</v-btn>
							<v-btn
								:color="liveMode ? 'success' : 'secondary'"
								variant="tonal"
								size="small"
								@click="toggleLiveMode"
								:prepend-icon="
									liveMode
										? 'wifi_tethering'
										: 'wifi_tethering_off'
								"
							>
								{{ liveMode ? 'Live On' : 'Live Off' }}
							</v-btn>
							<v-btn
								color="info"
								variant="tonal"
								size="small"
								@click="replayGroupFlow"
								:loading="replaying"
								prepend-icon="play_circle"
							>
								Replay Flow
							</v-btn>
							<v-tooltip
								:text="
									viewMode !== 'associations'
										? 'Connect Mode is only available in Devices view'
										: connectMode
											? 'Exit Connect Mode'
											: 'Enter Connect Mode'
								"
								location="bottom"
							>
								<template #activator="{ props }">
									<v-btn
										v-bind="props"
										:color="
											connectMode
												? 'success'
												: 'secondary'
										"
										variant="tonal"
										size="small"
										:disabled="viewMode !== 'associations'"
										@click="connectMode = !connectMode"
										prepend-icon="add_link"
									>
										{{
											connectMode
												? 'Connect On'
												: 'Connect'
										}}
									</v-btn>
								</template>
							</v-tooltip>
							<v-chip
								v-if="liveMode"
								color="success"
								size="small"
								prepend-icon="circle"
								class="animate-pulse-chip"
							>
								Listening for events
							</v-chip>
							<v-chip
								v-if="replaying"
								color="info"
								size="small"
								prepend-icon="circle"
								class="animate-pulse-chip"
							>
								{{ replayLabel }}
							</v-chip>
						</v-col>
					</v-row>
				</v-expansion-panel-text>
			</v-expansion-panel>
		</v-expansion-panels>

		<!-- Connect Mode banner -->
		<v-alert
			v-if="connectMode"
			type="info"
			variant="tonal"
			density="compact"
			closable
			class="mx-3 mt-2"
			icon="add_link"
			@click:close="connectMode = false"
		>
			<strong>Connect Mode:</strong> Hover a node to see its port handle
			(blue dot), then drag to another node to create an association.
			Click an existing edge (line) to delete it.
		</v-alert>

		<!-- Loading -->
		<v-col
			align-self="center"
			class="text-center"
			v-show="loading"
			cols="12"
		>
			<v-progress-circular
				:size="50"
				color="primary"
				indeterminate
				class="mt-4"
			/>
			<div class="mt-2 text-caption">
				Loading associations… {{ loadProgress }}
			</div>
		</v-col>

		<!-- Graph canvas -->
		<v-col
			class="fill-height"
			:style="{ opacity: loading ? 0 : '' }"
			cols="12"
			ref="container"
			v-resize="onResize"
		>
			<div :style="{ height: containerHeight + 'px' }" ref="content" />

			<!-- Hover tooltip -->
			<v-menu
				v-model="showMenu"
				:close-on-content-click="false"
				location="bottom left"
				:style="{
					position: 'fixed',
					left: menuX + 'px',
					top: menuY + 'px',
				}"
			>
				<v-card v-if="hoverItem" min-width="260">
					<v-list-subheader class="ml-2 font-weight-bold">
						{{ hoverItem.label }}
					</v-list-subheader>
					<v-divider />
					<v-list
						density="compact"
						style="background: transparent"
						class="pa-0 text-caption"
					>
						<template v-if="hoverItem.type === 'group'">
							<v-list-item density="compact">
								Group ID
								<template #append
									><strong>{{
										hoverItem.groupId
									}}</strong></template
								>
							</v-list-item>
							<v-list-item density="compact">
								Source Node
								<template #append
									><strong>{{
										hoverItem.sourceNodeName
									}}</strong></template
								>
							</v-list-item>
							<v-list-item density="compact">
								Members
								<template #append
									><strong>{{
										hoverItem.memberCount
									}}</strong></template
								>
							</v-list-item>
							<v-list-item density="compact">
								Max Nodes
								<template #append
									><strong>{{
										hoverItem.maxNodes
									}}</strong></template
								>
							</v-list-item>
							<v-list-item
								density="compact"
								v-if="hoverItem.isLifeline"
							>
								<template #prepend
									><v-icon color="amber" size="small"
										>star</v-icon
									></template
								>
								Lifeline group
							</v-list-item>
							<v-list-item
								density="compact"
								v-if="hoverItem.hasDeadMember"
							>
								<template #prepend
									><v-icon color="error" size="small"
										>warning</v-icon
									></template
								>
								<span style="color: #c62828"
									>Has dead/failed member</span
								>
							</v-list-item>
						</template>

						<template v-else-if="hoverItem.type === 'node'">
							<v-list-item density="compact">
								Node ID
								<template #append
									><strong>{{
										hoverItem.nodeId
									}}</strong></template
								>
							</v-list-item>
							<v-list-item density="compact">
								Status
								<template #append>
									<strong
										:style="{
											color: hoverItem.statusColor,
										}"
										>{{ hoverItem.status }}</strong
									>
								</template>
							</v-list-item>
							<v-list-item density="compact">
								Groups (source)
								<template #append
									><strong>{{
										hoverItem.groupCount
									}}</strong></template
								>
							</v-list-item>
							<v-list-item density="compact">
								Group memberships
								<template #append
									><strong>{{
										hoverItem.membershipCount
									}}</strong></template
								>
							</v-list-item>
						</template>
					</v-list>
				</v-card>
			</v-menu>
		</v-col>

		<!-- Association Management Panel -->
		<v-navigation-drawer
			v-model="showManagePanel"
			location="right"
			width="360"
			temporary
		>
			<v-toolbar density="compact" color="surface">
				<v-toolbar-title class="text-body-1 font-weight-bold">
					{{
						managePanelNode?.name ||
						managePanelNode?._name ||
						'Node ' + managePanelNode?.id
					}}
					<v-chip
						size="x-small"
						class="ml-1"
						color="secondary"
						variant="tonal"
						>ID {{ managePanelNode?.id }}</v-chip
					>
				</v-toolbar-title>
				<v-btn
					icon="close"
					variant="text"
					@click="showManagePanel = false"
				/>
			</v-toolbar>

			<v-list-subheader class="mt-2">
				Status:
				<strong
					:style="{
						color: managePanelNode
							? getNodeStatusColor(managePanelNode.id)
							: '',
					}"
				>
					{{
						managePanelNode ? getNodeStatus(managePanelNode.id) : ''
					}}
				</strong>
			</v-list-subheader>

			<v-divider />

			<div v-if="managePanelLoading" class="text-center pa-4">
				<v-progress-circular indeterminate color="primary" size="24" />
				<div class="text-caption mt-1">Loading associations…</div>
			</div>

			<template v-else-if="managePanelGroups.length">
				<v-list-subheader>
					Association Groups ({{ managePanelGroupsFiltered.length
					}}<span v-if="managePanelHiddenCount" class="text-disabled"
						>/{{ managePanelGroups.length }}</span
					>)
					<v-tooltip location="bottom">
						<template #activator="{ props }">
							<v-btn
								v-bind="props"
								:icon="
									showEmptyGroups
										? 'visibility'
										: 'visibility_off'
								"
								size="x-small"
								variant="text"
								class="ml-1"
								@click="showEmptyGroups = !showEmptyGroups"
							/>
						</template>
						{{
							showEmptyGroups
								? 'Hide empty groups'
								: `Show ${managePanelHiddenCount} empty group(s)`
						}}
					</v-tooltip>
					<v-btn
						icon="refresh"
						size="x-small"
						variant="text"
						class="ml-1"
						@click="refreshManagePanel"
					/>
				</v-list-subheader>

				<div
					v-if="!managePanelGroupsFiltered.length"
					class="text-center text-disabled text-caption pa-2"
				>
					All groups are empty.
					<a style="cursor: pointer" @click="showEmptyGroups = true"
						>Show them</a
					>
				</div>

				<v-expansion-panels variant="accordion" density="compact">
					<v-expansion-panel
						v-for="g in managePanelGroupsFiltered"
						:key="g.groupKey"
					>
						<v-expansion-panel-title
							density="compact"
							class="text-caption"
						>
							<v-icon
								:color="g.isLifeline ? 'amber' : 'primary'"
								size="small"
								class="mr-1"
								>{{ g.isLifeline ? 'star' : 'hub' }}</v-icon
							>
							{{ g.title }}
							<v-chip
								size="x-small"
								class="ml-2"
								variant="outlined"
							>
								{{ g.members.length }}/{{ g.maxNodes || '?' }}
							</v-chip>
							<v-chip
								v-if="g.hasDeadMember"
								size="x-small"
								color="error"
								class="ml-1"
								>dead</v-chip
							>
						</v-expansion-panel-title>

						<v-expansion-panel-text class="pa-0">
							<!-- Members list -->
							<v-list density="compact" class="pa-0">
								<v-list-item
									v-for="m in g.members"
									:key="`${m.nodeId}-${m.targetEndpoint}`"
									density="compact"
									class="pl-3 pr-1"
								>
									<template #prepend>
										<v-icon
											size="small"
											:color="
												isNodeDead(m.nodeId)
													? 'error'
													: 'success'
											"
											>circle</v-icon
										>
									</template>
									<v-list-item-title class="text-caption">
										{{ getNodeName(m.nodeId) }}
										<span class="text-disabled">
											#{{ m.nodeId }}</span
										>
										<span
											v-if="m.targetEndpoint > 0"
											class="text-disabled"
										>
											ep{{ m.targetEndpoint }}</span
										>
									</v-list-item-title>
									<template #append>
										<v-btn
											icon="close"
											size="x-small"
											variant="text"
											color="error"
											:loading="
												removeLoading[
													`${g.groupId}_${m.nodeId}_${m.targetEndpoint}`
												]
											"
											@click.stop="removeMember(g, m)"
										/>
									</template>
								</v-list-item>

								<v-list-item
									v-if="!g.members.length"
									density="compact"
									class="text-disabled text-caption pl-3"
								>
									No members
								</v-list-item>
							</v-list>

							<!-- Add member -->
							<div
								class="pa-2"
								v-if="!g.isLifeline || g.maxNodes > 1"
							>
								<v-autocomplete
									v-model="addTargetNode[g.groupKey]"
									:items="availableTargetNodes(g)"
									item-title="_name"
									item-value="id"
									label="Add node…"
									density="compact"
									variant="outlined"
									hide-details
									clearable
									class="text-caption"
									@update:model-value="addMember(g, $event)"
								/>
							</div>
						</v-expansion-panel-text>
					</v-expansion-panel>
				</v-expansion-panels>
			</template>

			<div v-else class="text-center text-disabled text-caption pa-4">
				No association groups found for this node.
			</div>

			<!-- Incoming associations: other devices that control this node -->
			<template v-if="managePanelIncoming.length">
				<v-divider class="my-1" />
				<v-list-subheader>
					Controlled by ({{ managePanelIncoming.length }})
				</v-list-subheader>
				<v-list density="compact" class="pa-0">
					<v-list-item
						v-for="inc in managePanelIncoming"
						:key="inc.nodeId"
						density="compact"
						class="pl-3"
					>
						<template #prepend>
							<v-icon
								size="small"
								:color="
									isNodeDead(inc.nodeId) ? 'error' : 'info'
								"
								>arrow_forward</v-icon
							>
						</template>
						<v-list-item-title class="text-caption">
							{{ getNodeName(inc.nodeId) }}
							<span class="text-disabled">
								#{{ inc.nodeId }}</span
							>
						</v-list-item-title>
						<v-list-item-subtitle class="text-caption">
							{{ inc.groups.map((g) => g.groupTitle).join(', ') }}
						</v-list-item-subtitle>
					</v-list-item>
				</v-list>
			</template>
		</v-navigation-drawer>

		<!-- Connect Mode: new association dialog (drag-to-connect) -->
		<DialogAssociation
			v-if="showConnectorDialog"
			v-model="showConnectorDialog"
			:node="connectorSource"
			:associations="
				connectorSource ? associationsMap[connectorSource.id] || [] : []
			"
			:lockedSource="connectorSource"
			:lockedTarget="connectorTarget"
			@add="onConnectorAdd"
			@close="showConnectorDialog = false"
		/>

		<!-- Connect Mode: delete association snackbar -->
		<v-snackbar v-model="showDeleteSnackbar" :timeout="5000" color="error">
			Remove association
			<strong>{{ getNodeName(pendingDeleteEdge?.sourceNodeId) }}</strong>
			→ <strong>{{ getNodeName(pendingDeleteEdge?.targetNodeId) }}</strong
			>?
			<template #actions>
				<v-btn variant="text" @click="confirmDeleteEdge">Remove</v-btn>
				<v-btn variant="text" @click="showDeleteSnackbar = false"
					>Cancel</v-btn
				>
			</template>
		</v-snackbar>
	</div>
</template>

<script>
import { Network } from 'vis-network'
import { DataSet } from 'vis-data'
import 'vis-network/styles/vis-network.css'
import { mapActions, mapState } from 'pinia'
import useBaseStore from '../../stores/base.js'
import InstancesMixin from '../../mixins/InstancesMixin.js'
import DialogAssociation from '../dialogs/DialogAssociation.vue'

const NODE_STATUS_COLORS = {
	Dead: '#8b0000',
	Failed: '#8b0000',
	Alive: '#2DCC70',
	Awake: '#2DCC70',
	Asleep: '#F1C40F',
	Unknown: '#666666',
}

const GROUP_COLOR = '#3F51B5'
const GROUP_DEAD_COLOR = '#8b0000'
const GROUP_LIFELINE_COLOR = '#7e57c2'
const SHARED_NODE_COLOR = '#00BCD4'

export default {
	name: 'AssociationGraph',
	components: { DialogAssociation },
	mixins: [InstancesMixin],
	props: {
		nodes: {
			type: Array,
			default: () => [],
		},
		socket: Object,
	},
	computed: {
		...mapState(useBaseStore, {
			controllerNode: 'controllerNode',
			nodesMap: 'nodesMap',
			storeAssociationsMap: 'associationsMap',
			storeAssociationsLoaded: 'associationsLoaded',
		}),
		fontColor() {
			return this.$vuetify.theme.current.dark ? '#ddd' : '#333'
		},
		isDark() {
			return this.$vuetify.theme.current.dark
		},
		managePanelIncoming() {
			if (!this.managePanelNode) return []
			const targetId = this.managePanelNode.id
			const result = []
			for (const [srcNodeIdStr, assocs] of Object.entries(
				this.associationsMap,
			)) {
				const srcNodeId = Number(srcNodeIdStr)
				if (srcNodeId === targetId) continue
				const matchingGroups = {}
				for (const a of assocs || []) {
					if (a.nodeId === targetId) {
						const key = `ep${a.endpoint}_g${a.groupId}`
						if (!matchingGroups[key]) {
							matchingGroups[key] = {
								endpoint: a.endpoint,
								groupId: a.groupId,
								groupTitle: this.getGroupTitle(
									srcNodeId,
									a.endpoint,
									a.groupId,
								),
							}
						}
					}
				}
				const groups = Object.values(matchingGroups)
				if (groups.length > 0) {
					result.push({ nodeId: srcNodeId, groups })
				}
			}
			return result
		},
		managePanelGroupsFiltered() {
			if (this.showEmptyGroups) return this.managePanelGroups
			return this.managePanelGroups.filter((g) => g.members.length > 0)
		},
		managePanelHiddenCount() {
			return (
				this.managePanelGroups.length -
				this.managePanelGroupsFiltered.length
			)
		},
	},
	network: null,
	_animationId: null,
	_demoInterval: null,
	data() {
		return {
			openPanel: -1,
			containerHeight: 400,
			loading: false,
			loadProgress: '',
			viewMode: 'associations',
			layoutMode: 'force',
			showDeadOnly: false,
			showLifelineGroups: false,
			showEmptyGroups: false,
			showMenu: false,
			menuX: 0,
			// Management panel
			showManagePanel: false,
			managePanelNode: null,
			managePanelGroups: [],
			managePanelLoading: false,
			addTargetNode: {},
			removeLoading: {},
			menuY: 0,
			hoverItem: null,
			hoverTimeout: null,
			showDetail: false,
			selectedDetail: null, // kept for group-view diamond clicks
			dragging: false,
			connectMode: false,
			connectorSource: null,
			connectorTarget: null,
			showConnectorDialog: false,
			pendingDeleteEdge: null,
			showDeleteSnackbar: false,
			demoMode: false,
			liveMode: false,
			replaying: false,
			replayLabel: '',
			pulses: [], // { from, to, progress, color, onComplete }
			// associationsMap: nodeId -> [{ endpoint, groupId, nodeId (member), targetEndpoint }]
			associationsMap: {},
			legends: [
				{
					color: GROUP_COLOR,
					textColor: GROUP_COLOR,
					text: 'Association Group',
					icon: 'hub',
				},
				{
					color: GROUP_LIFELINE_COLOR,
					textColor: GROUP_LIFELINE_COLOR,
					text: 'Lifeline Group',
					icon: 'star',
				},
				{
					color: GROUP_DEAD_COLOR,
					textColor: GROUP_DEAD_COLOR,
					text: 'Group with dead member',
					icon: 'warning',
				},
				{
					color: '#2DCC70',
					textColor: '#2DCC70',
					text: 'Alive Node',
					icon: 'radio_button_checked',
				},
				{
					color: '#F1C40F',
					textColor: '#F1C40F',
					text: 'Sleeping Node',
					icon: 'radio_button_checked',
				},
				{
					color: '#8b0000',
					textColor: '#8b0000',
					text: 'Dead / Failed Node',
					icon: 'cancel',
				},
				{
					color: SHARED_NODE_COLOR,
					textColor: SHARED_NODE_COLOR,
					text: 'Node in multiple groups',
					icon: 'share',
				},
			],
			stats: {
				totalGroups: 0,
				groupsWithDead: 0,
				sharedNodes: 0,
				totalAssociations: 0,
				deadNodes: 0,
			},
		}
	},
	watch: {
		connectMode(val) {
			val ? this.enableConnectMode() : this.disableConnectMode()
		},
		viewMode() {
			this.connectMode = false
			this.paintGraph()
		},
		layoutMode() {
			this.paintGraph()
		},
		showDeadOnly() {
			this.paintGraph()
		},
		showLifelineGroups() {
			this.paintGraph()
		},
		showEmptyGroups() {
			this.paintGraph()
		},
		nodes(val) {
			if (!val || val.length === 0) return
			if (Object.keys(this.associationsMap).length > 0) {
				this.paintGraph()
			} else if (Object.keys(this.storeAssociationsMap).length > 0) {
				Object.assign(this.associationsMap, this.storeAssociationsMap)
				this.paintGraph()
			} else if (!this.storeAssociationsLoaded && !this.loading) {
				// First-ever load: nodes arrived and nothing has been fetched yet
				this.loadAllAssociations()
			}
		},
	},
	mounted() {
		if (Object.keys(this.storeAssociationsMap).length > 0) {
			// Cache hit: use stored data, no API call
			Object.assign(this.associationsMap, this.storeAssociationsMap)
			this.$nextTick(() => this.paintGraph())
		} else if (
			!this.storeAssociationsLoaded &&
			this.nodes &&
			this.nodes.length > 0
		) {
			// First load: nothing fetched yet in this session
			this.loadAllAssociations()
		}
		// else: storeAssociationsLoaded=true but map is empty (e.g. after network reset)
		// — show empty state, user can hit Refresh
	},
	beforeUnmount() {
		this.stopAnimation()
		this.teardownLiveEvents()
		this.destroyNetwork()
		if (this.hoverTimeout) clearTimeout(this.hoverTimeout)
	},
	methods: {
		...mapActions(useBaseStore, ['setAssociations', 'clearAssociations']),
		enableConnectMode() {
			if (!this.network) return
			this.network.setOptions({
				manipulation: {
					enabled: true,
					initiallyActive: true,
					addNode: false,
					deleteNode: false,
					deleteEdge: false,
					addEdge: (data, callback) => {
						callback(null) // prevent vis from adding a real edge
						this.onConnectDrop(data.from, data.to)
					},
				},
			})
		},
		disableConnectMode() {
			if (!this.network) return
			this.network.setOptions({ manipulation: { enabled: false } })
			this.pendingDeleteEdge = null
			this.showDeleteSnackbar = false
		},
		onConnectDrop(fromVisId, toVisId) {
			if (fromVisId === toVisId) return
			// Guard: group diamond nodes don't start with 'node_'
			if (
				(typeof fromVisId === 'string' &&
					!fromVisId.startsWith('node_')) ||
				(typeof toVisId === 'string' && !toVisId.startsWith('node_'))
			)
				return
			const sourceId = parseInt(String(fromVisId).replace('node_', ''))
			const targetId = parseInt(String(toVisId).replace('node_', ''))
			const sourceNode = this.nodes[this.nodesMap.get(sourceId)]
			const targetNode = this.nodes[this.nodesMap.get(targetId)]
			if (!sourceNode || !targetNode) return
			this.connectorSource = sourceNode
			this.connectorTarget = targetNode
			this.showConnectorDialog = true
		},
		async onConnectorAdd(group) {
			if (!this.connectorSource || !this.connectorTarget) return
			const source = {
				nodeId: this.connectorSource.id,
				endpoint: group.endpoint ?? undefined,
			}
			const target = { nodeId: this.connectorTarget.id }
			const resp = await this.app.apiRequest('addAssociations', [
				source,
				group.group.value,
				[target],
			])
			this.showConnectorDialog = false
			if (resp.success) {
				const fresh = await this.app.apiRequest('getAssociations', [
					this.connectorSource.id,
					false,
				])
				if (fresh.success) {
					this.associationsMap[this.connectorSource.id] = fresh.result
					this.setAssociations(this.associationsMap)
				}
				this.paintGraph()
			}
		},
		async confirmDeleteEdge() {
			if (!this.pendingDeleteEdge) return
			const d = this.pendingDeleteEdge
			this.showDeleteSnackbar = false
			this.pendingDeleteEdge = null
			const source = {
				nodeId: d.sourceNodeId,
				endpoint: d.sourceEndpoint ?? undefined,
			}
			const target = {
				nodeId: d.targetNodeId,
				endpoint: d.targetEndpoint ?? undefined,
			}
			await this.app.apiRequest('removeAssociations', [
				source,
				d.groupId,
				[target],
			])
			const fresh = await this.app.apiRequest('getAssociations', [
				d.sourceNodeId,
				false,
			])
			if (fresh.success) {
				this.associationsMap[d.sourceNodeId] = fresh.result
				this.setAssociations(this.associationsMap)
			}
			this.paintGraph()
		},
		onResize() {
			this.containerHeight = this.$refs.container.$el.offsetHeight
			const maxHeight = window.innerHeight - 180
			if (this.containerHeight > maxHeight) {
				this.containerHeight = maxHeight
			}
		},
		destroyNetwork() {
			this.stopAnimation()
			if (this.network) {
				this.network.destroy()
				this.network = null
				if (this.$refs.content) this.$refs.content.innerHTML = ''
			}
			// Resolve any pending pulse promises so async flows (e.g. replayGroupFlow) don't hang
			for (const p of this.pulses) {
				if (p.onComplete) p.onComplete()
			}
			this.pulses = []
			this.replaying = false
		},
		stabilizeGraph() {
			if (!this.network) return
			this.network.setOptions({
				physics: {
					enabled: true,
					stabilization: { fit: false },
				},
			})
			this.network.once('stabilizationIterationsDone', () => {
				this.network.setOptions({ physics: false })
			})
			this.network.stabilize()
		},
		getNodeName(nodeId) {
			const idx = this.nodesMap.get(nodeId)
			const node = idx !== undefined ? this.nodes[idx] : null
			if (!node) return 'NodeID_' + nodeId
			return node.name || node._name || 'NodeID_' + nodeId
		},
		getNodeStatus(nodeId) {
			const idx = this.nodesMap.get(nodeId)
			const node = idx !== undefined ? this.nodes[idx] : null
			if (!node) return 'Unknown'
			if (node.failed || !node.available) return 'Dead'
			return node.status || 'Unknown'
		},
		getNodeStatusColor(nodeId) {
			const status = this.getNodeStatus(nodeId)
			return NODE_STATUS_COLORS[status] || '#666666'
		},
		getGroupTitle(nodeId, endpoint, groupId) {
			const idx = this.nodesMap.get(nodeId)
			const node = idx !== undefined ? this.nodes[idx] : null
			if (!node || !node.groups) return `Group ${groupId}`
			const g = node.groups.find(
				(g) => g.value === groupId && g.endpoint === endpoint,
			)
			return g?.title || `Group ${groupId}`
		},
		isNodeDead(nodeId) {
			const idx = this.nodesMap.get(nodeId)
			const node = idx !== undefined ? this.nodes[idx] : null
			if (!node) return false
			return node.failed || !node.available || node.status === 'Dead'
		},

		async loadAllAssociations() {
			this.loading = true
			this.clearAssociations()
			this.associationsMap = {}
			try {
				const nonControllerNodes = this.nodes.filter(
					(n) => !n.isControllerNode,
				)

				// Timeout sentinel so a hung socket call never blocks the whole load
				const withTimeout = (promise, ms) =>
					Promise.race([
						promise,
						new Promise((resolve) =>
							setTimeout(() => resolve({ success: false }), ms),
						),
					])

				for (let i = 0; i < nonControllerNodes.length; i++) {
					const node = nonControllerNodes[i]
					this.loadProgress = `${i + 1} / ${nonControllerNodes.length} nodes`
					try {
						const response = await withTimeout(
							this.app.apiRequest('getAssociations', [
								node.id,
								false,
							]),
							5000,
						)
						if (response.success) {
							this.associationsMap[node.id] = response.result
						}
					} catch {
						// node may not support associations
					}
				}

				// Persist to store so re-mounting the component uses cached data
				this.setAssociations(this.associationsMap)
				this.paintGraph()
			} finally {
				this.loading = false
				this.loadProgress = ''
			}
		},

		toggleDemoMode() {
			this.demoMode = !this.demoMode
			if (this.demoMode) {
				this.loadDemoAssociations()
				if (this.liveMode) this.startDemoSimulation()
			} else {
				this.stopDemoSimulation()
				// Restore from cache; only fetch if nothing was ever loaded
				if (Object.keys(this.storeAssociationsMap).length > 0) {
					Object.assign(
						this.associationsMap,
						this.storeAssociationsMap,
					)
					this.paintGraph()
				} else {
					this.loadAllAssociations()
				}
			}
		},

		loadDemoAssociations() {
			// Pick 6 real nodes to keep the demo small and easy to read.
			const nonCtrl = this.nodes.filter((n) => !n.isControllerNode)
			if (nonCtrl.length < 3) return

			// Take up to 6 evenly-spaced nodes from the real list
			const step = Math.max(1, Math.floor(nonCtrl.length / 6))
			const sample = []
			for (
				let i = 0;
				i < nonCtrl.length && sample.length < 6;
				i += step
			) {
				sample.push(nonCtrl[i])
			}

			// Fixed wiring: A→B, B→C, C→D, A→D, E→B, E→C (a small hub-spoke + ring)
			// Only these 6 nodes appear; every other node gets no associations.
			const [A, B, C, D, E, F] = sample
			const assoc = (nodeId, groupId, targetId) => ({
				endpoint: 0,
				groupId,
				nodeId: targetId,
				targetEndpoint: 0,
			})

			const demo = {}

			// All nodes report lifeline to controller
			for (const n of sample) {
				demo[n.id] = [assoc(n.id, 1, 1)]
			}

			// A controls B and D
			if (A && B) demo[A.id].push(assoc(A.id, 2, B.id))
			if (A && D) demo[A.id].push(assoc(A.id, 3, D.id))

			// B controls C
			if (B && C) demo[B.id].push(assoc(B.id, 2, C.id))

			// C loops back to A (creates a ring A→B→C→A)
			if (C && A) demo[C.id].push(assoc(C.id, 2, A.id))

			// E is a hub that controls both B and C
			if (E && B) demo[E.id].push(assoc(E.id, 2, B.id))
			if (E && C) demo[E.id].push(assoc(E.id, 3, C.id))

			// F bridges to D and E
			if (F && D) demo[F.id].push(assoc(F.id, 2, D.id))
			if (F && E) demo[F.id].push(assoc(F.id, 3, E.id))

			this.associationsMap = demo
			this.paintGraph()
		},

		// ── Live / animation ──────────────────────────────────────────────────

		toggleLiveMode() {
			this.liveMode = !this.liveMode
			if (this.liveMode) {
				this.setupLiveEvents()
				if (this.demoMode) this.startDemoSimulation()
			} else {
				this.teardownLiveEvents()
				this.stopDemoSimulation()
			}
		},

		setupLiveEvents() {
			if (!this.socket) return
			// Guard: never register the same handlers twice
			this.teardownLiveEvents()
			this.socket.on('VALUE_UPDATED', this._onValueUpdated)
			this.socket.on('NODE_EVENT', this._onNodeEvent)
		},

		teardownLiveEvents() {
			if (!this.socket) return
			this.socket.off('VALUE_UPDATED', this._onValueUpdated)
			this.socket.off('NODE_EVENT', this._onNodeEvent)
			this.stopDemoSimulation()
		},

		_onValueUpdated(data) {
			if (!this.liveMode || !this.network) return
			const nodeId = data?.nodeId ?? data?.id
			if (!nodeId) return
			// Throttle: Z-Wave fires many VALUE_UPDATED per action — one pulse per node per 500 ms
			if (!this.$options._pulseThrottle) this.$options._pulseThrottle = {}
			const now = Date.now()
			if (now - (this.$options._pulseThrottle[nodeId] ?? 0) < 500) return
			this.$options._pulseThrottle[nodeId] = now

			// Choose colour by commandClassName
			const cc = data?.commandClassName || ''
			const color = cc.includes('Binary')
				? '#00BCD4'
				: cc.includes('Multilevel')
					? '#F1C40F'
					: cc.includes('Notification')
						? '#FF7043'
						: '#2DCC70'
			this.spawnPulsesForNode(nodeId, color, cc || 'Value updated')
		},

		_onNodeEvent(data) {
			if (!this.liveMode || !this.network) return
			const nodeId = data?.nodeId ?? data?.id
			if (!nodeId) return
			this.spawnPulsesForNode(
				nodeId,
				'#7e57c2',
				data?.event || 'Node event',
			)
		},

		spawnPulsesForNode(nodeId, color, label) {
			if (!this.network) return
			const { groups } = this.buildGroupData()
			const srcId = `node_${nodeId}`

			const srcPos = this.getNodeCanvasPos(srcId)

			// Always flash the source node to give instant feedback
			if (srcPos) {
				this.pulses.push({
					type: 'flash',
					x: srcPos.x,
					y: srcPos.y,
					progress: 0,
					color,
				})
			}

			let pulsesAdded = 0

			// Groups owned by this node that have members → pulse outward
			const ownedGroups = groups.filter(
				(g) => g.nodeId === nodeId && g.members.length > 0,
			)
			for (const g of ownedGroups) {
				const from = this.getNodeCanvasPos(srcId)
				if (!from) continue

				const mid = this.getNodeCanvasPos(g.groupKey)

				if (!mid) {
					// Associations view — no diamonds: go direct node → each member
					for (const m of g.members) {
						const mPos = this.getNodeCanvasPos(`node_${m.nodeId}`)
						if (!mPos) continue
						pulsesAdded++
						this.pulses.push({
							from: { ...from },
							to: { ...mPos },
							progress: 0,
							color,
							label,
						})
					}
					continue
				}

				pulsesAdded++
				// Diamond view: Phase 1 node → group diamond
				this.pulses.push({
					from: { ...from },
					to: { ...mid },
					progress: 0,
					color,
					label,
					onComplete: () => {
						// Phase 2: group diamond → each member
						for (const m of g.members) {
							const mPos = this.getNodeCanvasPos(
								`node_${m.nodeId}`,
							)
							if (!mPos) continue
							this.pulses.push({
								from: { ...mid },
								to: { ...mPos },
								progress: 0,
								color,
								label,
							})
						}
					},
				})
			}

			// Groups this node is a MEMBER of → pulse inward (receiving)
			const memberGroups = groups.filter(
				(g) =>
					g.nodeId !== nodeId &&
					g.members.some((m) => m.nodeId === nodeId),
			)
			for (const g of memberGroups) {
				const grpPos = this.getNodeCanvasPos(g.groupKey)
				const mPos = this.getNodeCanvasPos(srcId)
				if (!grpPos || !mPos) continue
				pulsesAdded++
				this.pulses.push({
					from: { ...grpPos },
					to: { ...mPos },
					progress: 0,
					color: color + 'bb',
					label,
				})
			}

			// If the node isn't in the current view at all (srcPos null, pulsesAdded 0),
			// flash any group diamonds belonging to this node that ARE in the graph
			if (!srcPos && pulsesAdded === 0) {
				for (const g of ownedGroups) {
					const gPos = this.getNodeCanvasPos(g.groupKey)
					if (!gPos) continue
					this.pulses.push({
						type: 'flash',
						x: gPos.x,
						y: gPos.y,
						progress: 0,
						color,
					})
				}
			}
		},

		getNodeCanvasPos(visNodeId) {
			if (!this.network) return null
			try {
				return this.network.getPosition(visNodeId)
			} catch {
				return null
			}
		},

		startAnimation() {
			this.stopAnimation()
			const tick = () => {
				this.advancePulses()
				if (this.network && this.pulses.length > 0) {
					this.network.redraw()
				}
				this.$options._animationId = requestAnimationFrame(tick)
			}
			this.$options._animationId = requestAnimationFrame(tick)
		},

		stopAnimation() {
			if (this.$options._animationId) {
				cancelAnimationFrame(this.$options._animationId)
				this.$options._animationId = null
			}
		},

		advancePulses() {
			const TRAVEL_SPEED = 0.013
			const FLASH_SPEED = 0.025
			const next = []
			for (const p of this.pulses) {
				p.progress += p.type === 'flash' ? FLASH_SPEED : TRAVEL_SPEED
				if (p.progress >= 1) {
					if (p.onComplete) p.onComplete()
				} else {
					next.push(p)
				}
			}
			this.pulses = next
		},

		renderPulses(ctx) {
			if (!this.pulses.length) return
			ctx.save()
			for (const p of this.pulses) {
				if (p.type === 'flash') {
					// Expanding ring that fades out
					const radius = p.progress * 50
					const alpha = Math.max(0, 1 - p.progress)
					const hex = Math.floor(alpha * 255)
						.toString(16)
						.padStart(2, '0')
					ctx.beginPath()
					ctx.arc(p.x, p.y, radius, 0, Math.PI * 2)
					ctx.strokeStyle = p.color + hex
					ctx.lineWidth = 3 * (1 - p.progress)
					ctx.stroke()
					// Inner fill pulse
					const innerGrad = ctx.createRadialGradient(
						p.x,
						p.y,
						0,
						p.x,
						p.y,
						radius * 0.6,
					)
					innerGrad.addColorStop(
						0,
						p.color +
							Math.floor(alpha * 80)
								.toString(16)
								.padStart(2, '0'),
					)
					innerGrad.addColorStop(1, p.color + '00')
					ctx.beginPath()
					ctx.arc(p.x, p.y, radius * 0.6, 0, Math.PI * 2)
					ctx.fillStyle = innerGrad
					ctx.fill()
					continue
				}

				const t = p.progress
				const x = p.from.x + (p.to.x - p.from.x) * t
				const y = p.from.y + (p.to.y - p.from.y) * t

				// Trailing glow
				const grad = ctx.createRadialGradient(x, y, 0, x, y, 14)
				grad.addColorStop(0, p.color + 'ff')
				grad.addColorStop(0.4, p.color + '88')
				grad.addColorStop(1, p.color + '00')
				ctx.beginPath()
				ctx.arc(x, y, 14, 0, Math.PI * 2)
				ctx.fillStyle = grad
				ctx.fill()

				// Core dot
				ctx.beginPath()
				ctx.arc(x, y, 5, 0, Math.PI * 2)
				ctx.fillStyle = p.color
				ctx.shadowColor = p.color
				ctx.shadowBlur = 10
				ctx.fill()
				ctx.shadowBlur = 0
			}
			ctx.restore()
		},

		// Demo simulation — fires random pulses when both demoMode & liveMode are on
		startDemoSimulation() {
			this.stopDemoSimulation()

			// Only fire from the small demo node set (nodes that actually have
			// cross-associations in the current associationsMap)
			const activeNodeIds = Object.entries(this.associationsMap)
				.filter(([, assocs]) => assocs.some((a) => a.nodeId !== 1))
				.map(([id]) => parseInt(id))

			if (!activeNodeIds.length) return

			const COLORS = [
				'#00BCD4',
				'#F1C40F',
				'#FF7043',
				'#2DCC70',
				'#7e57c2',
			]
			const LABELS = [
				'Binary Switch',
				'Multilevel Set',
				'Scene Activation',
				'Notification',
				'Basic Set',
			]

			let idx = 0
			this.$options._demoInterval = setInterval(() => {
				if (!this.network) return
				// Cycle through active nodes in order so it's predictable
				const nodeId = activeNodeIds[idx % activeNodeIds.length]
				idx++
				const color = COLORS[idx % COLORS.length]
				const label = LABELS[idx % LABELS.length]
				this.spawnPulsesForNode(nodeId, color, label)
			}, 1800)
		},

		stopDemoSimulation() {
			if (this.$options._demoInterval) {
				clearInterval(this.$options._demoInterval)
				this.$options._demoInterval = null
			}
		},

		// ── Replay ────────────────────────────────────────────────────────────

		async replayGroupFlow() {
			if (this.replaying || !this.network) return
			this.replaying = true
			this.pulses = []

			const { groups } = this.buildGroupData()
			// Only groups with real non-controller members
			const active = groups.filter((g) =>
				g.members.some((m) => m.nodeId !== 1),
			)

			if (!active.length) {
				this.replaying = false
				return
			}

			const COLORS = [
				'#00BCD4',
				'#2DCC70',
				'#F1C40F',
				'#FF7043',
				'#7e57c2',
			]
			const delay = (ms) => new Promise((r) => setTimeout(r, ms))
			const PULSE_DURATION = 1000 / 0.013 // ~77 frames at 60fps ≈ 1.3s

			for (let i = 0; i < active.length; i++) {
				if (!this.replaying) break
				const g = active[i]
				const color = COLORS[i % COLORS.length]
				const srcName = this.getNodeName(g.nodeId)
				this.replayLabel = `${srcName} → ${g.title}`

				const srcPos = this.getNodeCanvasPos(`node_${g.nodeId}`)
				if (!srcPos) continue

				const grpPos = this.getNodeCanvasPos(g.groupKey)

				if (!grpPos) {
					// Associations / no-diamond view: go directly node → each member
					const memberPromises = g.members.map(
						(m) =>
							new Promise((resolve) => {
								const mPos = this.getNodeCanvasPos(
									`node_${m.nodeId}`,
								)
								if (!mPos) {
									resolve()
									return
								}
								this.pulses.push({
									from: { ...srcPos },
									to: { ...mPos },
									progress: 0,
									color,
									onComplete: resolve,
								})
							}),
					)
					await Promise.all(memberPromises)
					await delay(300)
					continue
				}

				// Diamond view: Phase 1 node → group diamond
				await new Promise((resolve) => {
					this.pulses.push({
						from: { ...srcPos },
						to: { ...grpPos },
						progress: 0,
						color,
						onComplete: resolve,
					})
				})

				// Phase 2: group diamond → each member (all at once)
				const memberPromises = g.members.map((m) => {
					return new Promise((resolve) => {
						const mPos = this.getNodeCanvasPos(`node_${m.nodeId}`)
						if (!mPos) {
							resolve()
							return
						}
						this.pulses.push({
							from: { ...grpPos },
							to: { ...mPos },
							progress: 0,
							color,
							onComplete: resolve,
						})
					})
				})
				await Promise.all(memberPromises)

				// Brief pause between groups
				await delay(300)
			}

			this.replayLabel = ''
			this.replaying = false
		},

		// ── Data ─────────────────────────────────────────────────────────────

		// Build a flat list of group objects:
		// { groupKey, nodeId, groupId, endpoint, title, maxNodes, multiChannel, isLifeline, members[] }
		buildGroupData() {
			const groups = []
			const groupMemberships = {} // nodeId -> [groupKey]

			for (const node of this.nodes) {
				if (node.isControllerNode) continue
				if (!node.groups || node.groups.length === 0) continue

				const nodeAssociations = this.associationsMap[node.id] || []

				for (const g of node.groups) {
					const groupKey = `n${node.id}_ep${g.endpoint}_g${g.value}`
					const members = nodeAssociations.filter(
						(a) =>
							a.groupId === g.value && a.endpoint === g.endpoint,
					)

					const isLifeline =
						g.title?.toLowerCase().includes('lifeline') ||
						g.value === 1

					// track memberships
					for (const m of members) {
						if (!groupMemberships[m.nodeId])
							groupMemberships[m.nodeId] = []
						if (!groupMemberships[m.nodeId].includes(groupKey)) {
							groupMemberships[m.nodeId].push(groupKey)
						}
					}

					groups.push({
						groupKey,
						nodeId: node.id,
						groupId: g.value,
						endpoint: g.endpoint,
						title: g.title || `Group ${g.value}`,
						maxNodes: g.maxNodes,
						multiChannel: g.multiChannel,
						isLifeline,
						members,
						hasDeadMember: members.some((m) =>
							this.isNodeDead(m.nodeId),
						),
					})
				}
			}

			return { groups, groupMemberships }
		},

		paintGraph() {
			this.destroyNetwork()

			if (!this.$refs.content) return

			const { groups, groupMemberships } = this.buildGroupData()

			// Compute stats
			this.stats.totalGroups = groups.length
			this.stats.groupsWithDead = groups.filter(
				(g) => g.hasDeadMember,
			).length
			this.stats.totalAssociations = groups.reduce(
				(sum, g) => sum + g.members.length,
				0,
			)
			this.stats.sharedNodes = Object.values(groupMemberships).filter(
				(m) => m.length > 1,
			).length
			this.stats.deadNodes = this.nodes.filter((n) =>
				this.isNodeDead(n.id),
			).length
			if (this.viewMode === 'associations') {
				this.paintAssociationsView(groups, groupMemberships)
				return
			}

			if (this.viewMode === 'groups') {
				this.paintGroupsView(groups, groupMemberships)
			} else if (this.viewMode === 'nodes') {
				this.paintNodesView(groups, groupMemberships)
			} else {
				this.paintSharedView(groups, groupMemberships)
			}
		},

		// ── Associations view (node → node, all devices shown) ───────────────

		paintAssociationsView(groups, groupMemberships) {
			const visNodes = new DataSet()
			const visEdges = new DataSet()

			// Add ALL Z-Wave nodes (so animation always has a position)
			const ctrlId = this.controllerNode?.id ?? 1
			for (const node of this.nodes) {
				const isDead = this.isNodeDead(node.id)
				const memberships = groupMemberships[node.id] || []
				const isShared = memberships.length > 1
				const statusColor = this.getNodeStatusColor(node.id)
				const color = isDead
					? '#8b0000'
					: node.isControllerNode
						? '#7e57c2'
						: isShared
							? SHARED_NODE_COLOR
							: statusColor
				const shape = node.isControllerNode
					? 'star'
					: node.isListening === false
						? 'square'
						: 'hexagon'
				visNodes.add({
					id: `node_${node.id}`,
					label: node.isControllerNode
						? 'Controller'
						: isDead
							? `⚠ ${this.getNodeName(node.id)}`
							: this.getNodeName(node.id),
					shape,
					color: {
						background: color,
						border: color,
						highlight: { background: color, border: '#fff' },
					},
					font: {
						color: isDead ? '#ff5252' : this.fontColor,
						size: 12,
						bold: isDead,
					},
					size: node.isControllerNode ? 22 : 16,
					borderWidth: isDead ? 3 : 1,
					_type: 'node',
					_nodeId: node.id,
					_status: this.getNodeStatus(node.id),
					_groupCount: node.groups?.length || 0,
					_groupMemberships: memberships,
				})
			}

			// Add one edge per association (one per group membership)
			for (const g of groups) {
				if (!this.showLifelineGroups && g.isLifeline) continue
				if (this.showDeadOnly && !g.hasDeadMember) continue
				for (const m of g.members) {
					const color = g.hasDeadMember
						? GROUP_DEAD_COLOR
						: g.isLifeline
							? '#888888'
							: '#00BCD4'
					visEdges.add({
						id: `${g.nodeId}_ep${g.endpoint}_g${g.groupId}_${m.nodeId}_ep${m.targetEndpoint ?? 0}`,
						from: `node_${g.nodeId}`,
						to: `node_${m.nodeId}`,
						title: g.title,
						dashes: g.isLifeline,
						color: {
							color,
							opacity: g.isLifeline ? 0.5 : 0.9,
						},
						arrows: { to: { enabled: true, scaleFactor: 0.6 } },
						width: 2,
						_type: 'association',
						_groups: [g],
						_assocData: {
							sourceNodeId: g.nodeId,
							sourceEndpoint: g.endpoint,
							groupId: g.groupId,
							targetNodeId: m.nodeId,
							targetEndpoint: m.targetEndpoint ?? 0,
						},
					})
				}
			}

			this.createNetwork(visNodes, visEdges)
		},

		// ── Association management ────────────────────────────────────────────

		async openManagePanel(nodeId) {
			const idx = this.nodesMap.get(nodeId)
			const node = idx !== undefined ? this.nodes[idx] : null
			if (!node) return

			this.managePanelNode = node
			this.managePanelGroups = []
			this.addTargetNode = {}
			this.managePanelLoading = true
			this.showManagePanel = true

			try {
				// Use refresh:false (controller cache) — no need to wake sleeping devices
				const resp = await this.app.apiRequest('getAssociations', [
					nodeId,
					false,
				])
				if (resp.success) {
					// Merge live associations into the node's group definitions
					const liveAssocs = resp.result || []
					const { groups } = this.buildGroupData()
					this.managePanelGroups = (node.groups || []).map((g) => {
						const groupKey = `n${nodeId}_ep${g.endpoint}_g${g.value}`
						const members = liveAssocs.filter(
							(a) =>
								a.groupId === g.value &&
								a.endpoint === g.endpoint,
						)
						const isLifeline =
							g.title?.toLowerCase().includes('lifeline') ||
							g.value === 1
						return {
							groupKey,
							groupId: g.value,
							endpoint: g.endpoint,
							title: g.title || `Group ${g.value}`,
							maxNodes: g.maxNodes,
							multiChannel: g.multiChannel,
							isLifeline,
							members,
							hasDeadMember: members.some((m) =>
								this.isNodeDead(m.nodeId),
							),
						}
					})
				}
			} catch {
				/* silently ignore */
			}
			this.managePanelLoading = false
		},

		async refreshManagePanel() {
			if (this.managePanelNode) {
				await this.openManagePanel(this.managePanelNode.id)
			}
		},

		availableTargetNodes(group) {
			if (!this.managePanelNode) return []
			const alreadyIn = new Set(group.members.map((m) => m.nodeId))
			return this.nodes.filter((n) => !alreadyIn.has(n.id))
		},

		async addMember(group, targetNodeId) {
			if (!targetNodeId || !this.managePanelNode) return
			const source = {
				nodeId: this.managePanelNode.id,
				endpoint: group.endpoint ?? undefined,
			}
			const target = { nodeId: targetNodeId }
			const resp = await this.app.apiRequest('addAssociations', [
				source,
				group.groupId,
				[target],
			])
			this.addTargetNode = {
				...this.addTargetNode,
				[group.groupKey]: null,
			}
			if (resp.success) {
				// Fetch fresh data once, update local cache, sync store, repaint, reopen panel
				const fresh = await this.app.apiRequest('getAssociations', [
					this.managePanelNode.id,
					false,
				])
				if (fresh.success) {
					this.associationsMap[this.managePanelNode.id] = fresh.result
					this.setAssociations(this.associationsMap)
				}
				this.paintGraph()
				await this.openManagePanel(this.managePanelNode.id)
			}
		},

		async removeMember(group, member) {
			if (!this.managePanelNode) return
			const key = `${group.groupId}_${member.nodeId}_${member.targetEndpoint}`
			this.removeLoading = { ...this.removeLoading, [key]: true }
			const source = {
				nodeId: this.managePanelNode.id,
				endpoint: group.endpoint ?? undefined,
			}
			const target = {
				nodeId: member.nodeId,
				endpoint:
					member.targetEndpoint >= 0
						? member.targetEndpoint
						: undefined,
			}
			let resp
			try {
				resp = await this.app.apiRequest('removeAssociations', [
					source,
					group.groupId,
					[target],
				])
			} finally {
				this.removeLoading = { ...this.removeLoading, [key]: false }
			}
			if (resp?.success) {
				const fresh = await this.app.apiRequest('getAssociations', [
					this.managePanelNode.id,
					false,
				])
				if (fresh.success) {
					this.associationsMap[this.managePanelNode.id] = fresh.result
					this.setAssociations(this.associationsMap)
				}
				this.paintGraph()
				await this.openManagePanel(this.managePanelNode.id)
			}
		},

		// View 1: Group-centric — group nodes connected to member nodes
		paintGroupsView(groups, groupMemberships) {
			const visNodes = new DataSet()
			const visEdges = new DataSet()
			const addedNodes = new Set()

			let filteredGroups = groups.filter((g) => {
				if (!this.showLifelineGroups && g.isLifeline) return false
				if (!this.showEmptyGroups && g.members.length === 0)
					return false
				if (this.showDeadOnly && !g.hasDeadMember) return false
				return true
			})

			for (const g of filteredGroups) {
				// Add group node
				const isLifeline = g.isLifeline
				const color = g.hasDeadMember
					? GROUP_DEAD_COLOR
					: isLifeline
						? GROUP_LIFELINE_COLOR
						: GROUP_COLOR

				visNodes.add({
					id: g.groupKey,
					label: `${this.getNodeName(g.nodeId)}\n${g.title}`,
					shape: 'diamond',
					color: {
						background: color,
						border: color,
						highlight: { background: color, border: '#fff' },
					},
					font: { color: this.fontColor, size: 11, multi: false },
					size: 20 + Math.min(g.members.length * 3, 20),
					// metadata for tooltip
					_type: 'group',
					_groupData: g,
					_sourceNodeName: this.getNodeName(g.nodeId),
				})

				// Add source node if not already there
				const srcId = `node_${g.nodeId}`
				if (!addedNodes.has(srcId)) {
					addedNodes.add(srcId)
					this.addNodeVis(visNodes, g.nodeId, groupMemberships)
				}

				// Edge: source node → group
				visEdges.add({
					from: srcId,
					to: g.groupKey,
					color: { color: color, opacity: 0.7 },
					dashes: false,
					arrows: { to: { enabled: true, scaleFactor: 0.6 } },
					width: 1.5,
					_type: 'source',
				})

				// Add member nodes and edges
				for (const m of g.members) {
					const mId = `node_${m.nodeId}`
					if (!addedNodes.has(mId)) {
						addedNodes.add(mId)
						this.addNodeVis(visNodes, m.nodeId, groupMemberships)
					}

					// Edge: group → member
					const edgeColor = this.isNodeDead(m.nodeId)
						? GROUP_DEAD_COLOR
						: color
					visEdges.add({
						from: g.groupKey,
						to: mId,
						color: { color: edgeColor, opacity: 0.85 },
						dashes: false,
						arrows: { to: { enabled: true, scaleFactor: 0.5 } },
						width: this.isNodeDead(m.nodeId) ? 2 : 1,
						_type: 'member',
					})
				}
			}

			this.createNetwork(visNodes, visEdges)
		},

		// View 2: Node-centric — nodes with edges to each group they belong to
		paintNodesView(groups, groupMemberships) {
			const visNodes = new DataSet()
			const visEdges = new DataSet()
			const addedGroups = new Set()
			const addedNodes = new Set()

			let filteredGroups = groups.filter((g) => {
				if (!this.showLifelineGroups && g.isLifeline) return false
				if (!this.showEmptyGroups && g.members.length === 0)
					return false
				return true
			})

			for (const g of filteredGroups) {
				if (!addedGroups.has(g.groupKey)) {
					addedGroups.add(g.groupKey)
					const color = g.hasDeadMember
						? GROUP_DEAD_COLOR
						: g.isLifeline
							? GROUP_LIFELINE_COLOR
							: GROUP_COLOR
					visNodes.add({
						id: g.groupKey,
						label: `${this.getNodeName(g.nodeId)}\n${g.title}`,
						shape: 'diamond',
						color: { background: color, border: color },
						font: { color: this.fontColor, size: 11 },
						size: 18,
						_type: 'group',
						_groupData: g,
						_sourceNodeName: this.getNodeName(g.nodeId),
					})
				}

				for (const m of g.members) {
					if (this.showDeadOnly && !this.isNodeDead(m.nodeId))
						continue
					const mId = `node_${m.nodeId}`
					if (!addedNodes.has(mId)) {
						addedNodes.add(mId)
						this.addNodeVis(visNodes, m.nodeId, groupMemberships)
					}
					const edgeColor = this.isNodeDead(m.nodeId)
						? GROUP_DEAD_COLOR
						: '#aaa'
					visEdges.add({
						from: mId,
						to: g.groupKey,
						color: { color: edgeColor, opacity: 0.7 },
						dashes: false,
						arrows: { to: { enabled: true, scaleFactor: 0.5 } },
						width: 1,
					})
				}
			}

			this.createNetwork(visNodes, visEdges)
		},

		// View 3: Shared-node view — highlight nodes that appear in multiple groups
		paintSharedView(groups, groupMemberships) {
			const visNodes = new DataSet()
			const visEdges = new DataSet()
			const addedNodes = new Set()

			// Only show groups that have shared members
			const sharedNodeIds = new Set(
				Object.entries(groupMemberships)
					.filter(([, m]) => m.length > 1)
					.map(([id]) => parseInt(id)),
			)

			for (const g of groups) {
				if (!this.showLifelineGroups && g.isLifeline) continue
				const sharedMembers = g.members.filter((m) =>
					sharedNodeIds.has(m.nodeId),
				)
				if (sharedMembers.length === 0 && !this.showEmptyGroups)
					continue

				const color = g.hasDeadMember
					? GROUP_DEAD_COLOR
					: g.isLifeline
						? GROUP_LIFELINE_COLOR
						: GROUP_COLOR

				if (!visNodes.get(g.groupKey)) {
					visNodes.add({
						id: g.groupKey,
						label: `${this.getNodeName(g.nodeId)}\n${g.title}`,
						shape: 'diamond',
						color: { background: color, border: color },
						font: { color: this.fontColor, size: 11 },
						size: 16,
						_type: 'group',
						_groupData: g,
						_sourceNodeName: this.getNodeName(g.nodeId),
					})
				}

				for (const m of sharedMembers) {
					const mId = `node_${m.nodeId}`
					if (!addedNodes.has(mId)) {
						addedNodes.add(mId)
						// Use SHARED_NODE_COLOR for shared nodes
						const nodeColor = this.isNodeDead(m.nodeId)
							? GROUP_DEAD_COLOR
							: SHARED_NODE_COLOR
						const node = this.nodes[this.nodesMap.get(m.nodeId)]
						visNodes.add({
							id: mId,
							label: this.getNodeName(m.nodeId),
							shape: 'hexagon',
							color: { background: nodeColor, border: nodeColor },
							font: { color: this.fontColor, size: 12 },
							size: 18,
							_type: 'node',
							_nodeId: m.nodeId,
							_groupMemberships: groupMemberships[m.nodeId] || [],
							_status: this.getNodeStatus(m.nodeId),
							_groupCount: node?.groups?.length || 0,
						})
					}
					visEdges.add({
						from: mId,
						to: g.groupKey,
						color: { color: SHARED_NODE_COLOR, opacity: 0.8 },
						width: 2,
						arrows: { to: { enabled: true, scaleFactor: 0.5 } },
					})
				}
			}

			this.createNetwork(visNodes, visEdges)
		},

		addNodeVis(visNodes, nodeId, groupMemberships) {
			const idx = this.nodesMap.get(nodeId)
			const node = idx !== undefined ? this.nodes[idx] : null
			const isDead = this.isNodeDead(nodeId)
			const memberships = groupMemberships[nodeId] || []
			const isShared = memberships.length > 1

			const nodeColor = isDead
				? GROUP_DEAD_COLOR
				: isShared
					? SHARED_NODE_COLOR
					: this.getNodeStatusColor(nodeId)

			const shape = node?.isListening === false ? 'square' : 'hexagon'

			visNodes.add({
				id: `node_${nodeId}`,
				label: isDead
					? `⚠ ${this.getNodeName(nodeId)}`
					: this.getNodeName(nodeId),
				shape,
				color: {
					background: nodeColor,
					border: nodeColor,
					highlight: { background: nodeColor, border: '#fff' },
				},
				font: {
					color: isDead ? '#ff5252' : this.fontColor,
					size: 12,
					bold: isDead,
				},
				size: 16,
				borderWidth: isDead ? 3 : 1,
				_type: 'node',
				_nodeId: nodeId,
				_groupMemberships: memberships,
				_status: this.getNodeStatus(nodeId),
				_groupCount: node?.groups?.length || 0,
			})
		},

		createNetwork(visNodes, visEdges) {
			if (!this.$refs.content) return

			// Ensure EVERY Z-Wave node has a registered position so live animation
			// always works, even for nodes hidden by filters (pure receivers, etc.)
			for (const node of this.nodes) {
				const vid = `node_${node.id}`
				if (!visNodes.get(vid)) {
					visNodes.add({
						id: vid,
						label: '',
						shape: 'dot',
						size: 4,
						hidden: true,
						physics: false,
						x: 0,
						y: 0,
						_type: 'anchor',
						_nodeId: node.id,
					})
				}
			}

			const isHierarchical = this.layoutMode === 'hierarchical'

			const options = {
				physics: {
					enabled: !isHierarchical,
					stabilization: {
						enabled: true,
						iterations: 300,
						updateInterval: 25,
						onlyDynamicEdges: false,
						fit: true,
					},
					barnesHut: {
						gravitationalConstant: -8000,
						centralGravity: 0.05,
						springLength: 180,
						springConstant: 0.03,
						damping: 0.15,
						avoidOverlap: 0.8,
					},
				},
				layout: isHierarchical
					? {
							hierarchical: {
								enabled: true,
								direction: 'UD',
								sortMethod: 'directed',
								levelSeparation: 120,
							},
						}
					: { hierarchical: false },
				interaction: {
					hover: true,
					navigationButtons: true,
					keyboard: true,
					multiselect: true,
					hideEdgesOnDrag: false,
					hideEdgesOnZoom: false,
					hideNodesOnDrag: false,
					hoverConnectedEdges: false,
					selectable: true,
					selectConnectedEdges: false,
					tooltipDelay: 0,
					zoomSpeed: 1,
					zoomView: true,
				},
				nodes: {
					borderWidth: 2,
					widthConstraint: { maximum: 180 },
					shadow: false,
				},
				edges: {
					width: 2,
					smooth: { type: 'dynamic' },
					selectionWidth: 2,
				},
			}

			this.network = new Network(
				this.$refs.content,
				{ nodes: visNodes, edges: visEdges },
				options,
			)

			this.network.once('stabilizationIterationsDone', () => {
				this.network.setOptions({ physics: { enabled: false } })
			})

			// Re-apply connect mode if it was active before a repaint
			if (this.connectMode) {
				this.enableConnectMode()
			}

			// Pulse rendering — draw on vis-network's own canvas each frame
			this.network.on('afterDrawing', (ctx) => {
				this.renderPulses(ctx)
			})

			this.startAnimation()

			this.network.on('dragStart', () => {
				this.dragging = true
				this.network.setOptions({ physics: true })
			})

			this.network.on('dragEnd', () => {
				this.dragging = false
				this.stabilizeGraph()
			})

			this.network.on('hoverNode', (params) => {
				if (this.dragging) return
				if (this.hoverTimeout) clearTimeout(this.hoverTimeout)
				const nodeData = visNodes.get(params.node)
				if (!nodeData) return

				this.menuX = params.event.clientX + 5
				this.menuY = params.event.clientY + 5

				if (nodeData._type === 'group') {
					const g = nodeData._groupData
					this.hoverItem = {
						type: 'group',
						label: nodeData.label.replace('\n', ' – '),
						groupId: g.groupId,
						sourceNodeName: nodeData._sourceNodeName,
						memberCount: g.members.length,
						maxNodes: g.maxNodes,
						isLifeline: g.isLifeline,
						hasDeadMember: g.hasDeadMember,
					}
				} else {
					const memberships = nodeData._groupMemberships || []
					this.hoverItem = {
						type: 'node',
						label: nodeData.label,
						nodeId: nodeData._nodeId,
						status: nodeData._status,
						statusColor: this.getNodeStatusColor(nodeData._nodeId),
						groupCount: nodeData._groupCount,
						membershipCount: memberships.length,
					}
				}
				this.showMenu = true
			})

			this.network.on('blurNode', () => {
				this.hoverTimeout = setTimeout(() => {
					this.showMenu = false
					this.hoverItem = null
				}, 200)
			})

			this.network.on('click', (params) => {
				if (params.nodes.length === 0) {
					this.showDetail = false
					return
				}

				const nodeData = visNodes.get(params.nodes[0])
				if (!nodeData) return

				// Open management panel for any node (including controller)
				if (nodeData._type !== 'group') {
					const nodeId = nodeData._nodeId
					if (nodeId) this.openManagePanel(nodeId)
					return
				}

				// Diamond group click (Groups view only) — show old detail sheet
				const g = nodeData._groupData
				this.selectedDetail = {
					type: 'group',
					label: nodeData.label.replace('\n', ' – '),
					groupId: g.groupId,
					sourceNodeName: nodeData._sourceNodeName,
					maxNodes: g.maxNodes,
					isLifeline: g.isLifeline,
					members: g.members,
				}
				this.showDetail = true
			})

			this.network.on('selectEdge', (params) => {
				if (!this.connectMode) return
				if (!params.edges.length) return
				const edgeData = visEdges.get(params.edges[0])
				if (!edgeData?._assocData) return
				this.pendingDeleteEdge = edgeData._assocData
				this.showDeleteSnackbar = true
			})

			this.network.on('deselectEdge', () => {
				if (!this.showDeleteSnackbar) return
				this.pendingDeleteEdge = null
				this.showDeleteSnackbar = false
			})
		},
	},
}
</script>

<style scoped>
.fill-height {
	height: 100%;
}

/* Hide vis-network's built-in manipulation toolbar — we use our own banner */
:deep(.vis-manipulation),
:deep(.vis-edit-mode) {
	display: none !important;
}

.animate-pulse-chip {
	animation: chip-pulse 1.5s ease-in-out infinite;
}

@keyframes chip-pulse {
	0%,
	100% {
		opacity: 1;
	}
	50% {
		opacity: 0.5;
	}
}
</style>
