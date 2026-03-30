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
								<v-btn value="groups" size="small"
									>Groups</v-btn
								>
								<v-btn value="nodes" size="small"
									>Node→Groups</v-btn
								>
								<v-btn value="shared" size="small"
									>Shared Nodes</v-btn
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

		<!-- Selected item detail panel -->
		<v-bottom-sheet v-model="showDetail" scrollable>
			<v-card>
				<v-card-title class="d-flex justify-space-between align-center">
					<span>{{ selectedDetail?.label }}</span>
					<v-btn
						icon="close"
						variant="text"
						@click="showDetail = false"
					/>
				</v-card-title>
				<v-card-text v-if="selectedDetail">
					<template v-if="selectedDetail.type === 'group'">
						<v-list density="compact">
							<v-list-item
								>Source Node:
								<strong>{{
									selectedDetail.sourceNodeName
								}}</strong></v-list-item
							>
							<v-list-item
								>Group ID:
								<strong>{{
									selectedDetail.groupId
								}}</strong></v-list-item
							>
							<v-list-item
								>Max Members:
								<strong>{{
									selectedDetail.maxNodes
								}}</strong></v-list-item
							>
							<v-list-item v-if="selectedDetail.isLifeline">
								<template #prepend
									><v-icon color="amber"
										>star</v-icon
									></template
								>
								Lifeline group
							</v-list-item>
						</v-list>
						<v-divider class="my-2" />
						<div class="text-subtitle-2 mb-1">
							Members ({{ selectedDetail.members?.length || 0 }})
						</div>
						<v-chip
							v-for="m in selectedDetail.members"
							:key="m.nodeId"
							class="mr-1 mb-1"
							:color="getNodeStatusColor(m.nodeId)"
							size="small"
						>
							{{ getNodeName(m.nodeId) }} ({{ m.nodeId }})
							<span
								v-if="
									m.endpoint !== undefined && m.endpoint >= 0
								"
							>
								ep{{ m.endpoint }}</span
							>
						</v-chip>
					</template>

					<template v-else-if="selectedDetail.type === 'node'">
						<v-list density="compact">
							<v-list-item
								>Node ID:
								<strong>{{
									selectedDetail.nodeId
								}}</strong></v-list-item
							>
							<v-list-item>
								Status:
								<strong
									:style="{
										color: getNodeStatusColor(
											selectedDetail.nodeId,
										),
									}"
									>{{ selectedDetail.status }}</strong
								>
							</v-list-item>
						</v-list>
						<v-divider class="my-2" />
						<div class="text-subtitle-2 mb-1">
							Groups defined by this node ({{
								selectedDetail.groups?.length || 0
							}})
						</div>
						<v-chip
							v-for="g in selectedDetail.groups"
							:key="g.groupKey"
							class="mr-1 mb-1"
							color="primary"
							variant="tonal"
							size="small"
						>
							{{ g.title }}
						</v-chip>
						<v-divider class="my-2" />
						<div class="text-subtitle-2 mb-1">
							Member of groups ({{
								selectedDetail.memberships?.length || 0
							}})
						</div>
						<v-chip
							v-for="m in selectedDetail.memberships"
							:key="m.groupKey"
							class="mr-1 mb-1"
							color="secondary"
							variant="tonal"
							size="small"
						>
							{{ m.sourceNodeName }} – {{ m.title }}
						</v-chip>
					</template>
				</v-card-text>
			</v-card>
		</v-bottom-sheet>
	</div>
</template>

<script>
import { Network } from 'vis-network'
import { DataSet } from 'vis-data'
import 'vis-network/styles/vis-network.css'
import { mapState } from 'pinia'
import useBaseStore from '../../stores/base.js'
import InstancesMixin from '../../mixins/InstancesMixin.js'

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
	name: 'GroupGraph',
	mixins: [InstancesMixin],
	props: {
		nodes: {
			type: Array,
			default: () => [],
		},
		socket: Object,
	},
	computed: {
		...mapState(useBaseStore, ['controllerNode', 'nodesMap']),
		fontColor() {
			return this.$vuetify.theme.current.dark ? '#ddd' : '#333'
		},
		isDark() {
			return this.$vuetify.theme.current.dark
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
			viewMode: 'groups',
			layoutMode: 'force',
			showDeadOnly: false,
			showLifelineGroups: true,
			showEmptyGroups: false,
			showMenu: false,
			menuX: 0,
			menuY: 0,
			hoverItem: null,
			hoverTimeout: null,
			showDetail: false,
			selectedDetail: null,
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
		viewMode() {
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
			if (
				val &&
				val.length > 0 &&
				Object.keys(this.associationsMap).length === 0
			) {
				this.loadAllAssociations()
			} else {
				this.paintGraph()
			}
		},
	},
	mounted() {
		if (this.nodes && this.nodes.length > 0) {
			this.loadAllAssociations()
		}
	},
	beforeUnmount() {
		this.stopAnimation()
		this.teardownLiveEvents()
		this.destroyNetwork()
		if (this.hoverTimeout) clearTimeout(this.hoverTimeout)
	},
	methods: {
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
			this.pulses = []
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
		isNodeDead(nodeId) {
			const idx = this.nodesMap.get(nodeId)
			const node = idx !== undefined ? this.nodes[idx] : null
			if (!node) return false
			return node.failed || !node.available || node.status === 'Dead'
		},

		async loadAllAssociations() {
			this.loading = true
			this.associationsMap = {}
			const nonControllerNodes = this.nodes.filter(
				(n) => !n.isControllerNode,
			)

			for (let i = 0; i < nonControllerNodes.length; i++) {
				const node = nonControllerNodes[i]
				this.loadProgress = `${i + 1} / ${nonControllerNodes.length} nodes`
				try {
					const response = await this.app.apiRequest(
						'getAssociations',
						[node.id, false],
					)
					if (response.success) {
						this.associationsMap[node.id] = response.result
					}
				} catch {
					// node may not support associations
				}
			}

			this.loading = false
			this.loadProgress = ''
			this.paintGraph()
		},

		toggleDemoMode() {
			this.demoMode = !this.demoMode
			if (this.demoMode) {
				this.loadDemoAssociations()
				if (this.liveMode) this.startDemoSimulation()
			} else {
				this.stopDemoSimulation()
				this.loadAllAssociations()
			}
		},

		loadDemoAssociations() {
			// Build rich synthetic associations using the real node list so
			// names / IDs remain accurate, only the cross-node wiring is fictional.
			const nonCtrl = this.nodes.filter((n) => !n.isControllerNode)
			if (nonCtrl.length < 3) return

			const demo = {}

			// Helper – pick N unique random nodes excluding `self`
			const pick = (self, n) => {
				const pool = nonCtrl
					.filter((x) => x.id !== self)
					.map((x) => x.id)
				const out = []
				while (out.length < n && pool.length > 0) {
					const idx = Math.floor(Math.random() * pool.length)
					out.push(pool.splice(idx, 1)[0])
				}
				return out
			}

			for (const node of nonCtrl) {
				const assocs = []
				const groups = node.groups || []

				// Group 1 – Lifeline always points to controller
				assocs.push({
					endpoint: 0,
					groupId: 1,
					nodeId: 1,
					targetEndpoint: 0,
				})

				// Remaining groups: wire 1-3 random peers into some of them
				for (const g of groups.slice(1, 6)) {
					// ~60 % chance a group has non-lifeline members
					if (Math.random() < 0.6) {
						const targets = pick(
							node.id,
							Math.ceil(Math.random() * 2),
						)
						for (const t of targets) {
							assocs.push({
								endpoint: g.endpoint ?? 0,
								groupId: g.value,
								nodeId: t,
								targetEndpoint: 0,
							})
						}
					}
				}

				demo[node.id] = assocs
			}

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

			// Groups owned by this node that have members → pulse outward
			const ownedGroups = groups.filter(
				(g) => g.nodeId === nodeId && g.members.length > 0,
			)
			for (const g of ownedGroups) {
				const from = this.getNodeCanvasPos(srcId)
				const mid = this.getNodeCanvasPos(g.groupKey)
				if (!from || !mid) continue

				// Phase 1: node → group diamond
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
				this.pulses.push({
					from: { ...grpPos },
					to: { ...mPos },
					progress: 0,
					color: color + 'bb',
					label,
				})
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
			const SPEED = 0.013
			const next = []
			for (const p of this.pulses) {
				p.progress += SPEED
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
			const nonCtrl = this.nodes.filter((n) => !n.isControllerNode)
			if (!nonCtrl.length) return

			const COLORS = [
				'#00BCD4',
				'#F1C40F',
				'#FF7043',
				'#2DCC70',
				'#7e57c2',
				'#FF4081',
			]
			const LABELS = [
				'Binary Switch',
				'Multilevel Set',
				'Scene Activation',
				'Notification',
				'Basic Set',
				'Central Scene',
			]

			this.$options._demoInterval = setInterval(() => {
				if (!this.network) return
				const node = nonCtrl[Math.floor(Math.random() * nonCtrl.length)]
				const color = COLORS[Math.floor(Math.random() * COLORS.length)]
				const label = LABELS[Math.floor(Math.random() * LABELS.length)]
				this.spawnPulsesForNode(node.id, color, label)
			}, 1200)
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
				const grpPos = this.getNodeCanvasPos(g.groupKey)
				if (!srcPos || !grpPos) continue

				// Phase 1: source node → group diamond
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

			if (this.viewMode === 'groups') {
				this.paintGroupsView(groups, groupMemberships)
			} else if (this.viewMode === 'nodes') {
				this.paintNodesView(groups, groupMemberships)
			} else {
				this.paintSharedView(groups, groupMemberships)
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
						dashes: this.isNodeDead(m.nodeId) ? [5, 5] : false,
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
						dashes: this.isNodeDead(m.nodeId),
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

			const isHierarchical = this.layoutMode === 'hierarchical'

			const options = {
				physics: {
					enabled: !isHierarchical,
					stabilization: { iterations: 150, fit: true },
					barnesHut: {
						gravitationalConstant: -8000,
						centralGravity: 0.3,
						springLength: 150,
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
					tooltipDelay: 0,
					hideEdgesOnDrag: true,
				},
				nodes: {
					borderWidth: 1,
					shadow: false,
				},
				edges: {
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

			// Pulse rendering — draw on vis-network's own canvas each frame
			this.network.on('afterDrawing', (ctx) => {
				this.renderPulses(ctx)
			})

			this.startAnimation()

			this.network.on('hoverNode', (params) => {
				if (this.hoverTimeout) clearTimeout(this.hoverTimeout)
				const nodeData = visNodes.get(params.node)
				if (!nodeData) return

				const pos = this.network.canvasToDOM(
					this.network.getPosition(params.node),
				)
				this.menuX = pos.x + 20
				this.menuY = pos.y + 20

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

				if (nodeData._type === 'group') {
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
				} else {
					const nodeId = nodeData._nodeId
					const idx = this.nodesMap.get(nodeId)
					const node = idx !== undefined ? this.nodes[idx] : null
					const { groups: allGroups } = this.buildGroupData()

					this.selectedDetail = {
						type: 'node',
						label: nodeData.label,
						nodeId,
						status: nodeData._status,
						groups:
							node?.groups?.map((g) => ({
								groupKey: `n${nodeId}_ep${g.endpoint}_g${g.value}`,
								title: g.title || `Group ${g.value}`,
							})) || [],
						memberships: allGroups
							.filter((g) =>
								g.members.some((m) => m.nodeId === nodeId),
							)
							.map((g) => ({
								groupKey: g.groupKey,
								sourceNodeName: this.getNodeName(g.nodeId),
								title: g.title,
							})),
					}
				}
				this.showDetail = true
			})
		},
	},
}
</script>

<style scoped>
.fill-height {
	height: 100%;
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
