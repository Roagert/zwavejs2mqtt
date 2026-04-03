<template>
	<div class="fill-height">
		<v-expansion-panels v-model="openPanel">
			<v-expansion-panel>
				<v-expansion-panel-title> Options </v-expansion-panel-title>
				<v-expansion-panel-text>
					<v-row>
						<v-col>
							<v-list-subheader>Legend</v-list-subheader>
							<v-list density="compact">
								<v-list-item
									v-for="(item, i) in legends"
									:key="i"
								>
									<template #prepend>
										<v-icon :color="item.color">{{
											item.icon || 'turned_in'
										}}</v-icon>
									</template>

									<v-list-item-title
										:style="{ color: item.textColor }"
									>
										{{ item.text }}</v-list-item-title
									>
								</v-list-item>
							</v-list>
						</v-col>
						<v-col>
							<v-list-subheader>Edges</v-list-subheader>
							<v-list density="compact">
								<v-list-item
									v-for="(item, i) in edgesLegend"
									:key="i"
								>
									<template #prepend>
										<v-icon :color="item.color">{{
											item.icon || 'turned_in'
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
						<v-col>
							<v-list-subheader>Filters</v-list-subheader>

							<v-autocomplete
								:items="locations"
								v-model="locationsFilter"
								multiple
								label="Locations filter"
								clearable
								chips
								closable-chips
								variant="solo"
							>
								<template #append>
									<v-btn
										v-tooltip:bottom="'Invert selection'"
										@click="
											invertLocationsFilter =
												!invertLocationsFilter
										"
										icon="loop"
										:color="
											invertLocationsFilter
												? 'primary'
												: ''
										"
										:class="
											invertLocationsFilter
												? 'border-primary'
												: ''
										"
									/>
								</template>
							</v-autocomplete>

							<v-autocomplete
								:items="allNodes"
								v-model="nodesFilter"
								multiple
								label="Nodes filter"
								clearable
								item-title="_name"
								item-value="id"
								chips
								closable-chips
								variant="solo"
							>
								<template #append>
									<v-btn
										v-tooltip:bottom="'Invert selection'"
										@click="
											invertNodesFilter =
												!invertNodesFilter
										"
										icon="loop"
										:color="
											invertNodesFilter ? 'primary' : ''
										"
										:class="
											invertNodesFilter
												? 'border-primary'
												: ''
										"
									/>
								</template>
							</v-autocomplete>

							<v-checkbox
								v-model="showReturnRoutes"
								label="Show return routes"
								:disabled="selectedNodes.length === 0"
							></v-checkbox>

							<v-checkbox
								v-model="showApplicationRoutes"
								label="Show priority routes"
								:disabled="selectedNodes.length === 0"
							></v-checkbox>

							<v-btn
								class="mt-2"
								:color="liveMode ? 'success' : 'secondary'"
								variant="tonal"
								size="small"
								@click="toggleLiveMode"
								:prepend-icon="
									liveMode
										? 'wifi_tethering'
										: 'wifi_tethering_off'
								"
								:disabled="!socket"
							>
								{{ liveMode ? 'Live On' : 'Live Off' }}
							</v-btn>

							<v-btn
								class="mt-2"
								:color="
									groupHighlightMode
										? 'cyan-darken-1'
										: 'secondary'
								"
								variant="tonal"
								size="small"
								@click="toggleGroupHighlightMode"
								prepend-icon="hub"
							>
								{{
									groupHighlightMode
										? 'Groups On'
										: 'Groups Off'
								}}
							</v-btn>

							<v-chip
								v-if="groupHighlightMode && !hasAssociationData"
								color="warning"
								size="small"
								prepend-icon="warning"
								class="mt-2"
							>
								Load association data in Association Graph first
							</v-chip>

							<div class="mt-4">
								<v-list-subheader class="pa-0"
									>Graph Mode</v-list-subheader
								>
								<v-btn-toggle
									v-model="graphMode"
									mandatory
									density="compact"
									color="primary"
									class="mt-1"
								>
									<v-btn value="routes" size="small"
										>Routes</v-btn
									>
									<v-btn value="locations" size="small"
										>Locations</v-btn
									>
								</v-btn-toggle>
							</div>

							<div class="mt-3">
								<v-list-subheader class="pa-0"
									>Layout</v-list-subheader
								>
								<v-btn-toggle
									v-model="assocLayoutMode"
									mandatory
									density="compact"
									color="primary"
									class="mt-1"
								>
									<v-btn value="force" size="small"
										>Default</v-btn
									>
									<v-btn
										value="hierarchical"
										size="small"
										prepend-icon="account_tree"
										>Hierarchy</v-btn
									>
								</v-btn-toggle>
							</div>
						</v-col>
					</v-row>
				</v-expansion-panel-text>
			</v-expansion-panel>
		</v-expansion-panels>

		<div class="mt-5" style="height: calc(100% - 95px)">
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
				></v-progress-circular>
			</v-col>
			<v-col
				class="fill-height"
				:style="{ opacity: loading ? 0 : '' }"
				cols="12"
				ref="container"
				v-resize="onResize"
			>
				<div
					:style="{
						height: containerHeight + 'px',
					}"
					ref="content"
				></div>
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
					<v-card v-if="hoverNode">
						<v-list-subheader class="ml-2 font-weight-bold">{{
							hoverNode._name
						}}</v-list-subheader>

						<v-divider></v-divider>

						<v-list
							style="min-width: 300px; background: transparent"
							density="compact"
							class="pa-0 text-caption"
						>
							<v-list-item density="compact">
								ID
								<template #append>
									<span class="align-end font-weight-bold">{{
										hoverNode.id
									}}</span>
								</template>
							</v-list-item>
							<v-list-item density="compact">
								Product
								<template #append>
									<span class="align-end font-weight-bold">{{
										hoverNode.productLabel +
										(hoverNode.productDescription
											? ' (' +
												hoverNode.productDescription +
												')'
											: '')
									}}</span>
								</template>
							</v-list-item>
							<v-list-item density="compact">
								Power
								<template #append>
									<span class="align-end font-weight-bold">{{
										hoverNode.minBatteryLevel
											? hoverNode.minBatteryLevel + '%'
											: 'MAIN'
									}}</span>
								</template>
							</v-list-item>
							<v-list-item density="compact">
								Neighbors
								<template #append>
									<span class="align-end font-weight-bold">{{
										hoverNode.neighbors.join(', ') || 'None'
									}}</span>
								</template>
							</v-list-item>
						</v-list>
					</v-card>
				</v-menu>
			</v-col>
		</div>
	</div>
</template>

<style></style>

<script>
import { Network } from 'vis-network'
import { DataSet } from 'vis-data'
import 'vis-network/styles/vis-network.css'
// when need to test this, just uncomment this line and find replace `this.nodes` with `testNodes`
// import fakeNodes from '@/assets/testNodes.json'
import {
	ProtocolDataRate,
	protocolDataRateToString,
	rssiToString,
	isRssiError,
	RouteKind,
} from '@zwave-js/core'
import { uuid, arraysEqual } from '../../lib/utils'
import useBaseStore from '../../stores/base.js'
import { mapState } from 'pinia'

const ReturnRouteKind = {
	PRIORITY: 20,
	CUSTOM: 21,
}

// Association view mode constants (ported from GroupGraph)
const GROUP_COLOR = '#00BCD4'
const GROUP_LIFELINE_COLOR = '#888888'
const GROUP_DEAD_COLOR = '#8b0000'
const SHARED_NODE_COLOR = '#7e57c2'

const ASSOC_NODE_STATUS_COLORS = {
	Dead: '#8b0000',
	Failed: '#8b0000',
	Alive: '#2DCC70',
	Awake: '#2DCC70',
	Asleep: '#F1C40F',
	Unknown: '#666666',
}

export default {
	props: {
		nodes: {
			type: [Array],
		},
		socket: {
			type: Object,
			default: null,
		},
	},
	computed: {
		...mapState(useBaseStore, [
			'controllerNode',
			'associationsMap',
			'nodesMap',
		]),
		content() {
			return this.$refs.content
		},
		fontColor() {
			return this.isDark ? '#ddd' : '#333'
		},
		isDark() {
			return this.$vuetify.theme.current.dark
		},
		locations() {
			// get unique locations array from nodes
			return this.allNodes.reduce((acc, node) => {
				if (node.loc && acc.indexOf(node.loc) === -1) {
					acc.push(node.loc)
				}
				return acc
			}, [])
		},
		filteredNodes() {
			return this.allNodes.filter((n) => {
				if (n.isControllerNode) {
					return true
				}

				let toAdd = false

				// check if node is in selected locations
				if (this.locationsFilter.length > 0) {
					if (this.invertLocationsFilter) {
						toAdd = !this.locationsFilter.includes(n.loc)
					} else {
						toAdd = this.locationsFilter.includes(n.loc)
					}
				}

				// if not in current locations, check if it's on selected nodes
				if (!toAdd && this.nodesFilter.length > 0) {
					if (this.invertNodesFilter) {
						toAdd = !this.nodesFilter.includes(n.id)
					} else {
						toAdd = this.nodesFilter.includes(n.id)
					}
				}

				return toAdd
			})
		},
		allNodes() {
			return this.nodes // replace this with `fakeNodes` when testing
		},
		hasAssociationData() {
			return (
				this.associationsMap &&
				Object.keys(this.associationsMap).length > 0
			)
		},
	},
	network: null, // do not make this reactive, see https://github.com/visjs/vis-network/issues/173#issuecomment-541435420
	unsubscribeUpdate: null, // pinia update action unsubscribe function
	_routesFetchedFor: new Set(), // de-duplicates route fetch requests per node
	_animationId: null,
	data() {
		return {
			openPanel: -1,
			hoverNodeTimeout: null,
			containerHeight: 400,
			graphMode: 'routes',
			assocLayoutMode: 'force',
			liveMode: false,
			groupHighlightMode: false,
			groupHighlightNodeId: null,
			pulses: [],
			selectedNodes: [],
			showReturnRoutes: true,
			showApplicationRoutes: true,
			menuX: 0,
			menuY: 0,
			showMenu: false,
			hoverNode: null,
			locationsFilter: [],
			invertLocationsFilter: false,
			nodesFilter: [],
			invertNodesFilter: false,
			refreshTimeout: null,
			updateTimeout: null,
			loading: false,
			priorityEdges: {}, // keeps track of the edges that should be shown in overview
			legends: [
				{
					color: '#7e57c2',
					textColor: '#7e57c2',
					text: 'Controller',
				},
				{
					color: '#00BCD4',
					textColor: '#00BCD4',
					text: '1 hop',
				},
				{
					color: '#2DCC70',
					textColor: '#2DCC70',
					text: '2 hops',
				},
				{
					color: '#F1C40F',
					textColor: '#F1C40F',
					text: '3 hops',
				},
				{
					color: '#E77E23',
					textColor: '#E77E23',
					text: '4 hops',
				},
				{
					color: '#8b0000',
					textColor: '#8b0000',
					text: 'Failed Node',
				},
				{
					color: '#666666',
					textColor: '#666666',
					text: 'Unknown',
				},
			],
			edgesLegend: [
				{
					icon: 'star',
					textColor: '',
					color: '#F1C40F',
					text: 'Priority route',
				},
				{
					icon: 'minimize',
					textColor: '',
					text: 'Last working route',
				},
				{
					icon: 'more_horiz',
					textColor: '',
					text: 'Next to last working route',
				},
				{
					color: '#8b0000',
					textColor: '#8b0000',
					text: protocolDataRateToString(ProtocolDataRate.ZWave_9k6),
				},
				{
					color: '#F1C40F',
					textColor: '#F1C40F',
					text: protocolDataRateToString(ProtocolDataRate.ZWave_40k),
				},
				{
					color: '#2DCC70',
					textColor: '#2DCC70',
					text: protocolDataRateToString(ProtocolDataRate.ZWave_100k),
				},
				{
					color: '#3F51B5',
					textColor: '#3F51B5',
					text: protocolDataRateToString(
						ProtocolDataRate.LongRange_100k,
					),
				},
				{
					color: '#666666',
					textColor: '#666666',
					text: 'Unknown',
				},
			],
		}
	},
	watch: {
		graphMode() {
			this.paintGraph()
		},
		assocLayoutMode() {
			this.paintGraph()
		},
		filteredNodes(val, oldVal) {
			if (!arraysEqual(val, oldVal)) {
				this.selectedNodes = val.map((n) => n.id)
				this.setSelection()
			}
		},
		showReturnRoutes(v) {
			// hide/show all edges with returnRoute = true
			if (
				this.network &&
				!this.loading &&
				this.selectedNodes.length > 0
			) {
				const { edges } = this.network.body.data

				const edgesToUpdate = []

				edges.forEach((e) => {
					if (e.routeKind >= ReturnRouteKind.PRIORITY) {
						edgesToUpdate.push({
							id: e.id,
							hidden: !v,
						})
					}
				})

				edges.update(edgesToUpdate)
			}
		},
		showApplicationRoutes(v) {
			// hide/show all edges with returnRoute = false
			if (
				this.network &&
				!this.loading &&
				this.selectedNodes.length > 0
			) {
				const { edges } = this.network.body.data

				const edgesToUpdate = []

				edges.forEach((e) => {
					if (e.routeKind === RouteKind.Application) {
						edgesToUpdate.push({
							id: e.id,
							hidden: !v,
						})
					}
				})

				edges.update(edgesToUpdate)
			}
		},
	},
	mounted() {
		this.paintGraph()

		this.unsubscribeUpdate = useBaseStore().$onAction(
			({ name, args, after }) => {
				if (name === 'updateMeshGraph') {
					if (this.updateTimeout) {
						clearTimeout(this.updateTimeout)
					}
					this.updateTimeout = setTimeout(
						this.onNodeUpdate.bind(this, args[0]),
						1000,
					)
				} else if (name === 'initNodes') {
					this.debounceRefresh()
				} else if (name === 'updateNode') {
					after(() => {
						this.onNodeUpdatedOrAdded(args[0])
					})
				} else if (name === 'removeNode') {
					this.onNodeRemoved(args[0])
				}
			},
		)
	},
	beforeUnmount() {
		if (this.refreshTimeout) {
			clearTimeout(this.refreshTimeout)
		}

		if (this.updateTimeout) {
			clearTimeout(this.updateTimeout)
		}

		if (this.hoverNodeTimeout) {
			clearTimeout(this.hoverNodeTimeout)
		}

		this.teardownLiveEvents()
		this.stopAnimation()
		this.destroyNetwork()

		this.unsubscribeUpdate()
	},
	methods: {
		onResize() {
			// when container resizes get its height and set content to that
			// so that the graph can be resized
			this.containerHeight = this.$refs.container.$el.offsetHeight
			const maxHeight = window.innerHeight - 180
			// prevent to grow bigger then window height
			if (this.containerHeight > maxHeight) {
				this.containerHeight = maxHeight
			}
		},
		destroyNetwork() {
			this.stopAnimation()
			this.pulses = []
			if (this.network) {
				this.network.destroy()
				this.network = null
				this.content.innerHTML = ''
			}
		},
		debounceRefresh() {
			if (this.refreshTimeout) {
				clearTimeout(this.refreshTimeout)
			}

			this.loading = true

			this.refreshTimeout = setTimeout(this.paintGraph.bind(this), 1000)
		},
		onNodeUpdate(node) {
			if (this.updateTimeout) {
				clearTimeout(this.updateTimeout)
			}

			if (this.network && !this.loading) {
				const { nodes, edges } = this.network.body.data

				// collect old edges for this node, keyed by signature
				const oldEdgesByKey = {} // key => [edge, ...]
				const allOtherEdges = {} // edgeId => [edge, ...]

				edges.forEach((e) => {
					if (e.routeOf === node.id) {
						const key = `${e.from}-${e.to}-${e.routeKind}`
						if (!oldEdgesByKey[key]) {
							oldEdgesByKey[key] = []
						}
						oldEdgesByKey[key].push(e)
					} else {
						const edgeId = this.getEdgeId(e)
						if (allOtherEdges[edgeId]) {
							allOtherEdges[edgeId].push(e)
						} else {
							allOtherEdges[edgeId] = [e]
						}
					}
				})

				// clear priorityEdges for this node's old edges
				for (const key in oldEdgesByKey) {
					for (const e of oldEdgesByKey[key]) {
						const edgeId = this.getEdgeId(e)
						if (this.priorityEdges[edgeId]?.id === e.id) {
							delete this.priorityEdges[edgeId]
						}
					}
				}

				// parse new edges
				const result = this.parseNode(node)

				// match new edges to old ones by signature, reuse IDs
				const edgesToUpdate = []
				const edgesToAdd = []
				const usedOldIds = new Set()

				for (const newEdge of result.edges) {
					const key = `${newEdge.from}-${newEdge.to}-${newEdge.routeKind}`
					const oldPool = oldEdgesByKey[key]
					const oldEdge =
						oldPool && oldPool.find((e) => !usedOldIds.has(e.id))

					if (oldEdge) {
						usedOldIds.add(oldEdge.id)
						newEdge.id = oldEdge.id
						edgesToUpdate.push(newEdge)
					} else {
						edgesToAdd.push(newEdge)
					}
				}

				// remove unmatched old edges
				const edgesToRemove = []
				for (const key in oldEdgesByKey) {
					for (const e of oldEdgesByKey[key]) {
						if (!usedOldIds.has(e.id)) {
							edgesToRemove.push(e.id)
						}
					}
				}

				// update priorityEdges for removed edges that were priorities
				for (const eId of edgesToRemove) {
					const removedEdge = edges.get(eId)
					if (!removedEdge) continue
					const edgeId = this.getEdgeId(removedEdge)

					// only recompute if the removed edge was the current priority
					if (
						this.priorityEdges[edgeId] &&
						this.priorityEdges[edgeId].id === eId
					) {
						const candidates = allOtherEdges[edgeId]
						if (candidates && candidates.length > 0) {
							this.priorityEdges[edgeId] = candidates.reduce(
								(prev, curr) =>
									prev.protocolDataRate >
									curr.protocolDataRate
										? prev
										: curr,
							)
						} else {
							delete this.priorityEdges[edgeId]
						}
					}
				}

				edges.remove(edgesToRemove)
				edges.update(edgesToUpdate)
				edges.add(edgesToAdd)
				nodes.update(result.node)

				// node has no routes yet, request them (once per node)
				const hadNoEdges = Object.keys(oldEdgesByKey).length === 0
				if (
					hadNoEdges &&
					result.edges.length === 0 &&
					!this._routesFetchedFor.has(node.id)
				) {
					this._routesFetchedFor.add(node.id)
					this.$emit('node-added', node)
				}

				const params = {
					nodes: this.selectedNodes,
				}

				this.network.setSelection(params)
				this.handleSelectNode(params, true)
			}
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
		onNodeUpdatedOrAdded(n) {
			if (!this.network || this.loading) return

			const { nodes, edges } = this.network.body.data

			const node = useBaseStore().getNode(n.id)
			if (!node) return

			if (nodes.get(n.id)) {
				// node already in graph — update its visual properties
				// save/restore priorityEdges because parseNode calls
				// parseRouteStats which overwrites entries with edge
				// objects that are never added to the DataSet
				const saved = { ...this.priorityEdges }
				const result = this.parseNode(node)
				this.priorityEdges = saved
				nodes.update(result.node)
				return
			}

			// new node — add it with edges and stabilize
			const result = this.parseNode(node)
			nodes.add(result.node)
			edges.add(result.edges)

			// notify parent to fetch routes for this node so edges can be rendered
			this._routesFetchedFor.add(node.id)
			this.$emit('node-added', node)
			this.stabilizeGraph()
		},
		onNodeRemoved(n) {
			if (!this.network || this.loading) return

			const { nodes, edges } = this.network.body.data
			if (!nodes.get(n.id)) return

			const edgesToRemove = []
			edges.forEach((e) => {
				if (e.routeOf === n.id || e.from === n.id || e.to === n.id) {
					edgesToRemove.push(e.id)
				}
			})

			for (const eId of edgesToRemove) {
				const edge = edges.get(eId)
				if (edge) {
					const edgeId = this.getEdgeId(edge)
					if (this.priorityEdges[edgeId]?.id === eId) {
						delete this.priorityEdges[edgeId]
					}
				}
			}

			edges.remove(edgesToRemove)
			nodes.remove(n.id)

			// deselect removed node and un-hide others
			const idx = this.selectedNodes.indexOf(n.id)
			if (idx !== -1) {
				this.selectedNodes.splice(idx, 1)
				const params = { nodes: this.selectedNodes }
				this.network.setSelection(params)
				this.handleSelectNode(params)
			}

			this.$emit('node-removed', n)
			this.stabilizeGraph()
		},
		getEdgeId(edge) {
			return `${edge.from}-${edge.to}`
		},
		getDataRateColor(dataRate) {
			switch (dataRate) {
				case ProtocolDataRate.ZWave_9k6:
					return '#8b0000'
				case ProtocolDataRate.ZWave_40k:
					return '#F1C40F'
				case ProtocolDataRate.ZWave_100k:
					return '#2DCC70'
				case ProtocolDataRate.LongRange_100k:
					return '#3F51B5'
				default:
					return '#666666'
			}
		},
		clearSelection() {
			if (this.network && !this.loading) {
				this.network.unselectAll()
				this.handleSelectNode({ nodes: [] })
			}
		},
		setSelection() {
			if (this.network && !this.loading) {
				const emptyFilters =
					this.nodesFilter.length === 0 &&
					this.locationsFilter.length === 0 &&
					!this.invertNodesFilter &&
					!this.invertLocationsFilter

				// check if all nodes are selected
				const all =
					this.filteredNodes.length === this.allNodes.length &&
					emptyFilters

				const params = {
					nodes: all ? [] : this.selectedNodes,
				}
				this.network.setSelection(params)
				this.handleSelectNode(params)
			}
		},
		paintGraph() {
			if (this.graphMode === 'locations') {
				this.destroyNetwork()
				this.paintLocationsView()
				return
			}

			// Default: routes mode (existing behavior unchanged)
			this.destroyNetwork()

			this.priorityEdges = {}
			this._routesFetchedFor = new Set()
			this.groupHighlightNodeId = null

			this.loading = true

			const { edges, nodes } = this.parseNodes()

			const container = this.content
			const data = {
				nodes,
				edges,
			}
			const options = {
				interaction: {
					// https://visjs.github.io/vis-network/docs/network/interaction.html#
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
					tooltipDelay: 1000,
					zoomSpeed: 1,
					zoomView: true,
				},
				nodes: {
					borderWidth: 2,
					widthConstraint: {
						maximum: 180,
					},
					// shadow: true,
				},
				edges: {
					width: 2,
					// shadow: true,
				},
				physics: {
					enabled: this.assocLayoutMode !== 'hierarchical',
					stabilization: {
						enabled: true,
						iterations: 50,
						updateInterval: 50,
						onlyDynamicEdges: false,
						fit: true,
					},
					barnesHut: {
						theta: 0.99,
						damping: 0.9,
						avoidOverlap: 0.15,
					},
				},
				layout:
					this.assocLayoutMode === 'hierarchical'
						? {
								hierarchical: {
									enabled: true,
									direction: 'UD',
									sortMethod: 'directed',
									levelSeparation: 120,
								},
							}
						: { hierarchical: false },
			}

			this.network = new Network(container, data, options)

			// event handlers
			// https://visjs.github.io/vis-network/docs/network/#Events
			if (this.assocLayoutMode === 'hierarchical') {
				// Hierarchical layout doesn't use physics so stabilizationIterationsDone never fires
				this.$nextTick(() => {
					this.loading = false
					this.network.fit()
					this.setSelection()
				})
			} else {
				this.network.once('stabilizationIterationsDone', () => {
					this.loading = false
					this.network.setOptions({ physics: false })
					this.setSelection()
				})
			}

			this.network.on('afterDrawing', (ctx) => {
				this.renderPulses(ctx)
			})
			this.startAnimation()

			this.network.on('click', this.handleClick.bind(this))

			this.network.on('hoverNode', this.handleHoverNode.bind(this))
			this.network.on('blurNode', this.handleBlurNode.bind(this))

			this.network.on('dragStart', this.handleDragStart.bind(this))
			this.network.on('dragEnd', this.handleDragEnd.bind(this))

			this.network.on('select', this.handleSelectNode.bind(this))
		},
		handleSelectNode(params, skipFit = false) {
			let { nodes: selectedNodes } = params

			const { edges, nodes } = this.network.body.data
			const repeaters = []

			const edgesToUpdate = []
			const nodesToUpdate = []

			// click on controller
			if (
				selectedNodes.length === 1 &&
				nodes.get(selectedNodes[0]).isControllerNode
			) {
				selectedNodes = []
			}

			this.selectedNodes = selectedNodes

			const showAll = selectedNodes.length === 0

			// DataSet: https://visjs.github.io/vis-data/data/dataset.html
			edges.forEach((e) => {
				// Skip group overlay edges — they are managed independently
				if (e.isGroupOverlay) return

				const edgeId = this.getEdgeId(e)
				const shouldBeHidden =
					(showAll && this.priorityEdges[edgeId]?.id !== e.id) ||
					(selectedNodes.length > 0 &&
						!selectedNodes.includes(e.routeOf))

				let checkboxHide = false

				if (!showAll) {
					if (
						e.routeKind === RouteKind.Application &&
						!this.showApplicationRoutes
					) {
						checkboxHide = true
					} else if (
						e.routeKind >= ReturnRouteKind.PRIORITY &&
						!this.showReturnRoutes
					) {
						checkboxHide = true
					}
				}

				const fontSize = showAll ? 0 : 12

				if (shouldBeHidden !== e.hidden || fontSize !== e.font.size) {
					edgesToUpdate.push({
						id: e.id,
						hidden: shouldBeHidden || checkboxHide,
						font: {
							size: fontSize,
						},
					})
				}

				if (!shouldBeHidden) {
					repeaters.push(e.from)
					repeaters.push(e.to)
				}
			})

			edges.update(edgesToUpdate)

			nodes.forEach((n) => {
				const shouldBeHidden =
					selectedNodes.length > 0 &&
					!n.isControllerNode &&
					!selectedNodes.includes(n.id) &&
					!repeaters.includes(n.id)

				if (shouldBeHidden !== n.hidden) {
					nodesToUpdate.push({
						id: n.id,
						hidden: shouldBeHidden,
						color: n.color,
					})
				}
			})

			nodes.update(nodesToUpdate)

			if (!skipFit) {
				this.network.fit()
			}
		},
		handleDragStart() {
			this.dragging = true
			this.network.setOptions({ physics: true })
		},
		handleDragEnd() {
			this.dragging = false
			this.stabilizeGraph()
		},
		handleHoverNode(params) {
			// show menu
			if (this.dragging) return
			const { node, event } = params

			const item = this.allNodes.find((n) => n.id === node)

			if (item) {
				this.hoverNodeTimeout = setTimeout(() => {
					this.hoverNode = item
					this.menuX = event.clientX + 5
					this.menuY = event.clientY + 5
					this.showMenu = true
					this.hoverNodeTimeout = null
				}, 1000)
			}
		},
		handleBlurNode() {
			if (this.hoverNodeTimeout) {
				clearTimeout(this.hoverNodeTimeout)
				this.hoverNodeTimeout = null
			} else {
				// hide menu
				this.showMenu = false
				this.hoverNode = null
			}
		},
		handleClick(params) {
			if (params.event) {
				params.event.preventDefault()
				const nodeId = params.nodes[0]
				if (nodeId) {
					const node = this.allNodes.find((n) => n.id === nodeId)
					this.$emit('node-click', node)
				} else {
					this.$emit('node-click', null)
				}
				// sync graph visibility when the 'select' event
				// doesn't fire (e.g. clicking an already-selected
				// node after drag). Skip when selection changed,
				// since the 'select' event handler already covers it.
				const clickedNodes = params.nodes || []
				const selectedSet = new Set(this.selectedNodes)
				const selectionUnchanged =
					clickedNodes.length === selectedSet.size &&
					clickedNodes.every((id) => selectedSet.has(id))
				if (selectionUnchanged) {
					this.handleSelectNode(params, true)
				}

				// Group highlight overlay
				if (this.groupHighlightMode) {
					if (
						nodeId &&
						!this.allNodes.find((n) => n.id === nodeId)
							?.isControllerNode
					) {
						this.applyGroupHighlight(nodeId)
					} else {
						this.clearGroupOverlay()
						this.groupHighlightNodeId = null
					}
				}
			}
		},
		// ── Group highlight overlay ───────────────────────────────────────────

		toggleGroupHighlightMode() {
			this.groupHighlightMode = !this.groupHighlightMode
			if (!this.groupHighlightMode) {
				this.clearGroupOverlay()
				this.groupHighlightNodeId = null
			}
		},

		clearGroupOverlay() {
			if (!this.network) return
			const { nodes, edges } = this.network.body.data
			const toRemove = []
			edges.forEach((e) => {
				if (e.isGroupOverlay) toRemove.push(e.id)
			})
			if (toRemove.length) edges.remove(toRemove)

			// restore any dimmed nodes
			const toRestore = []
			nodes.forEach((n) => {
				if (n._groupDimmed) {
					toRestore.push({
						id: n.id,
						opacity: 1.0,
						_groupDimmed: false,
					})
				}
			})
			if (toRestore.length) nodes.update(toRestore)
		},

		applyGroupHighlight(clickedNodeId) {
			if (!this.network || !this.hasAssociationData) return

			this.clearGroupOverlay()
			this.groupHighlightNodeId = clickedNodeId

			const { nodes, edges } = this.network.body.data

			// Collect all nodes that share at least one association with clickedNodeId
			// (nodes clickedNodeId sends to, and nodes that send to clickedNodeId)
			const overlayEdgeDefs = [] // { fromId, toId, isLifeline }
			const relatedNodeIds = new Set()

			// Forward associations: clickedNode → targets
			const ownAssocs = this.associationsMap[String(clickedNodeId)] || []
			for (const assoc of ownAssocs) {
				const targetId = assoc.nodeId
				if (targetId == null) continue
				const isLifeline = assoc.groupId === 1
				overlayEdgeDefs.push({
					fromId: clickedNodeId,
					toId: targetId,
					isLifeline,
				})
				relatedNodeIds.add(targetId)
			}

			// Reverse associations: senders → clickedNode
			for (const [sourceIdStr, assocs] of Object.entries(
				this.associationsMap,
			)) {
				const sourceId = parseInt(sourceIdStr)
				if (sourceId === clickedNodeId) continue
				for (const assoc of assocs) {
					if (assoc.nodeId === clickedNodeId) {
						const isLifeline = assoc.groupId === 1
						overlayEdgeDefs.push({
							fromId: sourceId,
							toId: clickedNodeId,
							isLifeline,
						})
						relatedNodeIds.add(sourceId)
						break
					}
				}
			}

			// Dim unrelated nodes (not the clicked node, not related)
			const toDim = []
			nodes.forEach((n) => {
				if (n.id === clickedNodeId || n.isControllerNode) return
				const shouldDim = !relatedNodeIds.has(n.id)
				if (shouldDim && !n._groupDimmed) {
					toDim.push({ id: n.id, opacity: 0.25, _groupDimmed: true })
				} else if (!shouldDim && n._groupDimmed) {
					toDim.push({ id: n.id, opacity: 1.0, _groupDimmed: false })
				}
			})
			if (toDim.length) nodes.update(toDim)

			// Deduplicate overlay edges by (from, to, isLifeline)
			const seen = new Set()
			const overlayEdges = []
			for (const def of overlayEdgeDefs) {
				const key = `${def.fromId}_${def.toId}_${def.isLifeline}`
				if (seen.has(key)) continue
				seen.add(key)
				const color = def.isLifeline ? '#FFC107' : '#00BCD4'
				overlayEdges.push({
					id: `group_overlay_${def.fromId}_${def.toId}_${def.isLifeline ? 'L' : 'A'}`,
					from: def.fromId,
					to: def.toId,
					color: { color, opacity: 0.85 },
					width: 3,
					dashes: false,
					arrows: { to: { enabled: true, scaleFactor: 0.7 } },
					smooth: { enabled: true, type: 'dynamic' },
					font: { size: 0 },
					hidden: false,
					physics: false,
					isGroupOverlay: true,
				})
			}

			if (overlayEdges.length) edges.add(overlayEdges)
		},

		// ── /Group highlight overlay ──────────────────────────────────────────

		parseRouteStats(
			edges,
			controllerId,
			node,
			route,
			routeKind,
			forceShow = false,
		) {
			if (!route) {
				if (routeKind !== RouteKind.NLWR) {
					// unknown route
					node.color = this.legends[6].color
				}
				return
			}

			const isReturn = routeKind >= 20

			// tells if this route should be shown in overview
			const showInOverview =
				forceShow || (routeKind !== RouteKind.NLWR && !isReturn)

			const { repeaters, repeaterRSSI, rssi, routeFailedBetween } = route

			let { protocolDataRate } = route

			if (route.routeSpeed) {
				protocolDataRate = route.routeSpeed
			}

			if (routeFailedBetween) {
				const edge = {
					id: uuid(),
					from: routeFailedBetween[0],
					to: routeFailedBetween[1],
					color: this.getDataRateColor(ProtocolDataRate.ZWave_9k6),
					width: 1,
					label: 'Failed ❌',
					font: { align: 'top', size: 0 },
					dashes: [2, 2],
					hidden: true,
					routeOf: node.id, // used to know this edge needs to be shown when highlighting a node
					physics: true,
				}

				edges.push(edge)
			}

			const sourceNodeId = isReturn ? node.id : controllerId
			const destinationNodeId = isReturn ? controllerId : node.id

			for (let i = 0; i <= repeaters.length; i++) {
				const repeater = repeaters[i]
				const prevRepeater = repeaters[i - 1] || sourceNodeId

				const from = prevRepeater
				const to = repeater || destinationNodeId

				const edgeRssi = i === 0 ? rssi : repeaterRSSI?.[i - 1]

				let label = ''

				if (edgeRssi && !isRssiError(edgeRssi)) {
					label = rssiToString(edgeRssi)
				}

				const edgeId = this.getEdgeId({ from, to })

				const starArrow = {
					enabled: true,
					type: 'image',
					// don't use path here, seems vis-network doens't respect X-External-Path header causing 404 on HA Addon (issue https://github.com/zwave-js/zwave-js-ui/issues/3492)
					src: 'data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSIxNyIgaGVpZ2h0PSIxNSIgdmlld0JveD0iLTEgLTEgMTcgMTYiPgogIDxwYXRoIGQ9Im03LjUgMCAyLjI0IDQuNiA1IC43NS0zLjYyIDMuNTkuODYgNS4wNi00LjQ4LTIuNEwzLjAyIDE0bC44Ni01LjA2TC4yNiA1LjM1bDUtLjc0Wm0wIDAiIHN0eWxlPSJzdHJva2U6IzAwMDtmaWxsLXJ1bGU6bm9uemVybztmaWxsOiNmZmM5MDE7ZmlsbC1vcGFjaXR5OjEiLz4KPC9zdmc+',
					scaleFactor: 1,
				}

				let width, dashes, arrows

				switch (routeKind) {
					case RouteKind.NLWR:
						width = 1
						dashes = [5, 5]
						break
					case RouteKind.LWR:
						width = 4
						dashes = false
						break
					case RouteKind.Application:
						width = 4
						dashes = false
						arrows = {
							middle: starArrow,
						}
						break
					case ReturnRouteKind.PRIORITY:
						width = 2
						dashes = [5, 10]
						arrows = {
							to: {
								enabled: true,
								scaleFactor: 1,
							},
							middle: starArrow,
						}
						break
					case ReturnRouteKind.CUSTOM:
						width = 1
						dashes = [5, 10]
						arrows = {
							to: {
								enabled: true,
								scaleFactor: 1,
							},
						}
						break
					default:
						width = 1
						dashes = [5, 5]
				}

				// create the edge
				// https://visjs.github.io/vis-network/docs/network/edges.html
				const edge = {
					id: uuid(),
					from,
					to,
					color: this.getDataRateColor(protocolDataRate),
					width,
					layer: i + 1,
					label,
					font: {
						align: 'top',
						size: 0,
						vadjust: routeKind === RouteKind.Application ? -5 : 0,
					}, //  multi: 'html'
					// arrows: 'to from',
					dashes,
					arrows,
					hidden: !showInOverview,
					routeOf: node.id, // used to know this edge needs to be shown when highlighting a node
					physics: true,
					routeKind,
					protocolDataRate,
				}

				edges.push(edge)

				if (showInOverview) {
					if (!node.failed && node.available) {
						node.color = this.legends[repeaters.length + 1].color
					}

					// only draw the edge with higher data rate
					if (this.priorityEdges[edgeId]) {
						if (
							this.priorityEdges[edgeId].protocolDataRate >=
							protocolDataRate
						) {
							edge.hidden = true
						} else {
							this.priorityEdges[edgeId].hidden = true
							this.priorityEdges[edgeId] = edge
						}
					} else {
						this.priorityEdges[edgeId] = edge
					}
				}
			}
		},
		parseNodes() {
			const result = {
				edges: [],
				nodes: [],
			}

			for (const node of this.allNodes) {
				const { node: entity } = this.parseNode(node, result.edges)
				result.nodes.push(entity)
			}

			return result
		},

		// ── Association view helpers (ported from GroupGraph) ─────────────────

		_assocGetNodeName(nodeId) {
			const idx = this.nodesMap.get(nodeId)
			const node = idx !== undefined ? this.allNodes[idx] : null
			if (!node) return 'NodeID_' + nodeId
			return node.name || node._name || 'NodeID_' + nodeId
		},

		_assocGetNodeStatus(nodeId) {
			const idx = this.nodesMap.get(nodeId)
			const node = idx !== undefined ? this.allNodes[idx] : null
			if (!node) return 'Unknown'
			if (node.failed || !node.available) return 'Dead'
			return node.status || 'Unknown'
		},

		_assocGetNodeStatusColor(nodeId) {
			const status = this._assocGetNodeStatus(nodeId)
			return ASSOC_NODE_STATUS_COLORS[status] || '#666666'
		},

		_assocIsNodeDead(nodeId) {
			const idx = this.nodesMap.get(nodeId)
			const node = idx !== undefined ? this.allNodes[idx] : null
			if (!node) return false
			return node.failed || !node.available || node.status === 'Dead'
		},

		paintLocationsView() {
			const visNodes = new DataSet()
			const visEdges = new DataSet()

			// Palette for location cluster nodes
			const palette = [
				'#00BCD4',
				'#7E57C2',
				'#26A69A',
				'#EF5350',
				'#FF7043',
				'#29B6F6',
				'#66BB6A',
				'#FFA726',
				'#EC407A',
				'#8D6E63',
			]
			const locationColors = {}
			let colorIdx = 0
			const getLocationColor = (loc) => {
				const key = loc || '(No location)'
				if (!locationColors[key]) {
					locationColors[key] = palette[colorIdx % palette.length]
					colorIdx++
				}
				return locationColors[key]
			}

			// Add location cluster nodes
			const locationGroups = {}
			for (const node of this.allNodes) {
				const loc = node.loc || '(No location)'
				if (!locationGroups[loc]) locationGroups[loc] = []
				locationGroups[loc].push(node)
			}
			for (const loc of Object.keys(locationGroups)) {
				const color = getLocationColor(loc)
				visNodes.add({
					id: `loc_${loc}`,
					label: loc,
					shape: 'ellipse',
					color: {
						background: color,
						border: color,
						highlight: { background: color, border: '#fff' },
					},
					font: { color: '#fff', size: 13, bold: true },
					size: 24,
					_type: 'location',
				})
			}

			// Add device nodes using the exact same visual as Routes view.
			// parseNode calls parseRouteStats which writes into this.priorityEdges —
			// save and restore so Locations view doesn't corrupt the routes graph state.
			const savedPriorityEdges = { ...this.priorityEdges }
			for (const node of this.allNodes) {
				const loc = node.loc || '(No location)'
				const color = getLocationColor(loc)
				const { node: entity } = this.parseNode(node, [])
				visNodes.add(entity)

				visEdges.add({
					from: `loc_${loc}`,
					to: node.id,
					color: { color, opacity: 0.5 },
					dashes: false,
					width: 1,
					arrows: { to: { enabled: false } },
				})
			}
			this.priorityEdges = savedPriorityEdges

			this._createAssocNetwork(visNodes, visEdges)
		},

		_createAssocNetwork(visNodes, visEdges) {
			if (!this.content) return

			const isHierarchical = this.assocLayoutMode === 'hierarchical'

			const options = {
				physics: {
					enabled: !isHierarchical,
					stabilization: {
						enabled: true,
						iterations: 50,
						updateInterval: 50,
						onlyDynamicEdges: false,
						fit: true,
					},
					barnesHut: {
						theta: 0.99,
						damping: 0.9,
						avoidOverlap: 0.15,
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

			this.loading = true
			this.network = new Network(
				this.content,
				{ nodes: visNodes, edges: visEdges },
				options,
			)

			if (isHierarchical) {
				this.$nextTick(() => {
					this.loading = false
					this.network.fit()
				})
			} else {
				this.network.once('stabilizationIterationsDone', () => {
					this.loading = false
					this.network.setOptions({ physics: { enabled: false } })
				})
			}

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
		},

		// ── /Association view helpers ─────────────────────────────────────────
		// ── Live pulse animation ──────────────────────────────────────────────

		toggleLiveMode() {
			this.liveMode = !this.liveMode
			if (this.liveMode) {
				this.setupLiveEvents()
			} else {
				this.teardownLiveEvents()
			}
		},

		setupLiveEvents() {
			if (!this.socket) return
			// Guard: never register twice
			this.teardownLiveEvents()
			this.socket.on('VALUE_UPDATED', this._onValueUpdated)
			this.socket.on('NODE_EVENT', this._onNodeEvent)
		},

		teardownLiveEvents() {
			if (!this.socket) return
			this.socket.off('VALUE_UPDATED', this._onValueUpdated)
			this.socket.off('NODE_EVENT', this._onNodeEvent)
		},

		_onValueUpdated(data) {
			if (!this.liveMode || !this.network) return
			const nodeId = data?.nodeId ?? data?.id
			if (!nodeId) return
			if (nodeId === (this.controllerNode?.id ?? 1)) return
			const cc = data?.commandClassName || ''
			const color = cc.includes('Binary')
				? '#00BCD4'
				: cc.includes('Multilevel')
					? '#F1C40F'
					: cc.includes('Notification')
						? '#FF7043'
						: '#2DCC70'
			this.spawnPulseToController(nodeId, color)
		},

		_onNodeEvent(data) {
			if (!this.liveMode || !this.network) return
			const nodeId = data?.nodeId ?? data?.id
			if (!nodeId) return
			if (nodeId === (this.controllerNode?.id ?? 1)) return
			this.spawnPulseToController(nodeId, '#7e57c2')
		},

		spawnPulseToController(nodeId, color) {
			if (!this.network) return
			const controllerId = this.controllerNode?.id ?? 1
			// Throttle: one pulse per node per 400 ms
			if (!this.$options._pulseThrottle) this.$options._pulseThrottle = {}
			const now = Date.now()
			if (now - (this.$options._pulseThrottle[nodeId] ?? 0) < 400) return
			this.$options._pulseThrottle[nodeId] = now

			let from, to
			try {
				from = this.network.getPosition(nodeId)
				to = this.network.getPosition(controllerId)
			} catch {
				return
			}
			if (!from || !to) return
			this.pulses.push({
				from: { ...from },
				to: { ...to },
				progress: 0,
				color,
			})
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
			const SPEED = 0.015
			const next = []
			for (const p of this.pulses) {
				p.progress += SPEED
				if (p.progress < 1) next.push(p)
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

				const grad = ctx.createRadialGradient(x, y, 0, x, y, 12)
				grad.addColorStop(0, p.color + 'ff')
				grad.addColorStop(0.4, p.color + '88')
				grad.addColorStop(1, p.color + '00')
				ctx.beginPath()
				ctx.arc(x, y, 12, 0, Math.PI * 2)
				ctx.fillStyle = grad
				ctx.fill()

				ctx.beginPath()
				ctx.arc(x, y, 4, 0, Math.PI * 2)
				ctx.fillStyle = p.color
				ctx.shadowColor = p.color
				ctx.shadowBlur = 8
				ctx.fill()
				ctx.shadowBlur = 0
			}
			ctx.restore()
		},

		// ── /Live pulse animation ─────────────────────────────────────────────

		parseNode(node, edges = []) {
			const hubNode = this.controllerNode?.id ?? 1

			const id = node.id

			let batlev = node.minBatteryLevel

			const nodeName = node.name || 'NodeID ' + node.id

			// create node
			// https://visjs.github.io/vis-network/docs/network/nodes.html
			const entity = {
				id: id,
				hidden: false,
				label: nodeName,
				neighbors: node.neighbors,
				battery_level: batlev,
				group: node.loc,
				failed: node.failed,
				available: node.available,
				font: { color: this.fontColor },
				forwards:
					node.isControllerNode ||
					(node.ready && !node.failed && node.isListening),
			}

			if (id === hubNode) {
				entity.shape = 'star'
				entity.isControllerNode = true
				entity.color = this.legends[0].color
			} else if (node.isListening) {
				entity.shape = 'hexagon'
			} else {
				entity.shape = 'square'
			}

			if (node.failed) {
				entity.label = 'FAILED: ' + entity.label
				entity.group = 'Failed'
				entity.color = this.legends[5].color
			}

			if (!node.available) {
				entity.label = 'DEAD: ' + entity.label
				entity.group = 'Dead'
				entity.color = this.legends[5].color
			}

			if (hubNode === id) {
				entity.label = 'Controller'
			} else {
				// parse application route
				this.parseRouteStats(
					edges,
					hubNode,
					entity,
					node.applicationRoute,
					RouteKind.Application,
				)

				// parse node LWR (last working route) https://zwave-js.github.io/node-zwave-js/#/api/node?id=quotstatistics-updatedquot
				this.parseRouteStats(
					edges,
					hubNode,
					entity,
					node.statistics?.lwr,
					RouteKind.LWR,
				)

				// parse node NLWR (next to last working route)
				this.parseRouteStats(
					edges,
					hubNode,
					entity,
					node.statistics?.nlwr,
					RouteKind.NLWR,
					!node.statistics?.lwr,
				)

				if (node.customSUCReturnRoutes) {
					for (const r of node.customSUCReturnRoutes) {
						this.parseRouteStats(
							edges,
							hubNode,
							entity,
							r,
							ReturnRouteKind.CUSTOM,
						)
					}
				}

				if (node.prioritySUCReturnRoute) {
					this.parseRouteStats(
						edges,
						hubNode,
						entity,
						node.prioritySUCReturnRoute,
						ReturnRouteKind.PRIORITY,
					)
				}
			}

			return { node: entity, edges }
		},
	},
}
</script>
