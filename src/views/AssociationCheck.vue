<template>
	<v-container fluid>
		<v-row>
			<v-col>
				<v-card>
					<v-card-title class="d-flex align-center">
						<v-icon start>link_off</v-icon>
						Association Health Check
						<v-spacer />
						<v-btn
							:loading="loading"
							color="primary"
							prepend-icon="search"
							variant="tonal"
							@click="runCheck"
						>
							Run Check
						</v-btn>
					</v-card-title>
					<v-card-subtitle v-if="lastRun">
						Last checked: {{ lastRun }} — scanned
						{{ scannedCount }} nodes
					</v-card-subtitle>

					<v-card-text>
						<v-alert
							v-if="error"
							type="error"
							variant="tonal"
							class="mb-4"
						>
							{{ error }}
						</v-alert>

						<v-alert
							v-else-if="checked && issues.length === 0"
							type="success"
							variant="tonal"
							prepend-icon="check_circle"
						>
							No bad associations found — your network looks
							clean.
						</v-alert>

						<template v-else-if="issues.length > 0">
							<v-alert
								type="warning"
								variant="tonal"
								class="mb-4"
							>
								Found {{ issues.length }} bad
								{{
									issues.length === 1
										? 'association'
										: 'associations'
								}}. Removing them will not affect normal device
								operation.
							</v-alert>

							<v-table>
								<thead>
									<tr>
										<th>Source Node</th>
										<th>Group</th>
										<th>Target Node</th>
										<th>Reason</th>
										<th class="text-right">Action</th>
									</tr>
								</thead>
								<tbody>
									<tr v-for="(issue, i) in issues" :key="i">
										<td>
											<strong>{{
												issue.sourceNodeId
											}}</strong>
											<span
												class="text-medium-emphasis ml-1"
											>
												{{ issue.sourceName }}
											</span>
										</td>
										<td>{{ issue.groupId }}</td>
										<td>
											<strong>{{
												issue.targetNodeId
											}}</strong>
											<span
												v-if="
													issue.targetEndpoint != null
												"
												class="text-medium-emphasis"
											>
												ep{{ issue.targetEndpoint }}
											</span>
										</td>
										<td>
											<v-chip
												:color="
													issue.reason.includes(
														'failed',
													)
														? 'warning'
														: 'error'
												"
												size="small"
												variant="tonal"
											>
												{{ issue.reason }}
											</v-chip>
										</td>
										<td class="text-right">
											<v-btn
												:loading="!!removing[i]"
												color="error"
												size="small"
												variant="tonal"
												prepend-icon="delete"
												@click="
													removeAssociation(issue, i)
												"
											>
												Remove
											</v-btn>
										</td>
									</tr>
								</tbody>
							</v-table>
						</template>

						<v-alert
							v-else-if="!checked"
							type="info"
							variant="tonal"
						>
							Click <strong>Run Check</strong> to scan all nodes
							for associations pointing to missing or failed
							devices.
						</v-alert>
					</v-card-text>
				</v-card>
			</v-col>
		</v-row>
	</v-container>
</template>

<script>
import { mapState } from 'pinia'
import useBaseStore from '../stores/base.js'

export default {
	name: 'AssociationCheckView',

	props: {
		socket: Object,
	},

	data() {
		return {
			loading: false,
			checked: false,
			issues: [],
			removing: {},
			error: null,
			lastRun: null,
			scannedCount: 0,
		}
	},

	computed: {
		...mapState(useBaseStore, ['nodes', 'nodesMap']),
	},

	methods: {
		getNodeById(id) {
			const store = useBaseStore()
			return store.nodes[store.nodesMap.get(id)]
		},

		async runCheck() {
			if (!this.socket) {
				this.error = 'Socket not connected'
				return
			}

			this.loading = true
			this.error = null
			this.issues = []
			this.checked = false

			try {
				const nodeIds = this.nodes.map((n) => n.id)
				const found = []

				await Promise.all(
					nodeIds.map(
						(nodeId) =>
							new Promise((resolve) => {
								this.socket.emit(
									'ZWAVE_API',
									{ api: 'getAssociations', args: [nodeId] },
									(res) => {
										if (
											res.success &&
											Array.isArray(res.result)
										) {
											const sourceNode =
												this.getNodeById(nodeId)
											for (const assoc of res.result) {
												const target = this.getNodeById(
													assoc.nodeId,
												)
												if (!target) {
													found.push({
														sourceNodeId: nodeId,
														sourceName:
															sourceNode?.name ??
															`Node ${nodeId}`,
														groupId: assoc.groupId,
														targetNodeId:
															assoc.nodeId,
														targetEndpoint:
															assoc.targetEndpoint,
														reason: 'Target node does not exist in network',
													})
												} else if (target.failed) {
													found.push({
														sourceNodeId: nodeId,
														sourceName:
															sourceNode?.name ??
															`Node ${nodeId}`,
														groupId: assoc.groupId,
														targetNodeId:
															assoc.nodeId,
														targetEndpoint:
															assoc.targetEndpoint,
														reason: 'Target node is marked as failed',
													})
												}
											}
										}
										resolve()
									},
								)
							}),
					),
				)

				this.issues = found
				this.scannedCount = nodeIds.length
				this.checked = true
				this.lastRun = new Date().toLocaleTimeString()
			} catch (err) {
				this.error = err.message
			} finally {
				this.loading = false
			}
		},

		removeAssociation(issue, index) {
			if (!this.socket) return
			this.removing = { ...this.removing, [index]: true }
			this.socket.emit(
				'ZWAVE_API',
				{
					api: 'removeAssociations',
					args: [
						{ nodeId: issue.sourceNodeId, endpoint: 0 },
						issue.groupId,
						[
							{
								nodeId: issue.targetNodeId,
								endpoint: issue.targetEndpoint,
							},
						],
					],
				},
				(res) => {
					const next = { ...this.removing }
					delete next[index]
					this.removing = next
					if (res.success) {
						this.issues = this.issues.filter((_, i) => i !== index)
					} else {
						this.error = `Failed to remove: ${res.message}`
					}
				},
			)
		},
	},
}
</script>
