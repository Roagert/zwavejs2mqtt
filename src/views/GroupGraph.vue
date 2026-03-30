<template>
	<v-container class="fill" fluid>
		<association-graph
			ref="groupGraph"
			id="group-graph"
			:nodes="nodes"
			:socket="socket"
		/>
	</v-container>
</template>

<script>
import { defineAsyncComponent } from 'vue'
import { mapState } from 'pinia'
import useBaseStore from '../stores/base.js'

export default {
	name: 'GroupGraphView',
	props: {
		socket: Object,
	},
	components: {
		AssociationGraph: defineAsyncComponent(
			() => import('@/components/custom/GroupGraph.vue'),
		),
	},
	computed: {
		...mapState(useBaseStore, ['nodes']),
	},
}
</script>
