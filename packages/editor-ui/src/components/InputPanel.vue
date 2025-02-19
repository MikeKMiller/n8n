<template>
	<RunData
		:nodeUi="currentNode"
		:runIndex="runIndex"
		:linkedRuns="linkedRuns"
		:canLinkRuns="canLinkRuns"
		:tooMuchDataTitle="$locale.baseText('ndv.input.tooMuchData.title')"
		:noDataInBranchMessage="$locale.baseText('ndv.input.noOutputDataInBranch')"
		:isExecuting="isExecutingPrevious"
		:executingMessage="$locale.baseText('ndv.input.executingPrevious')"
		:sessionId="sessionId"
		:overrideOutputs="connectedCurrentNodeOutputs"
		paneType="input"
		@linkRun="onLinkRun"
		@unlinkRun="onUnlinkRun"
		@runChange="onRunIndexChange">
		<template v-slot:header>
			<div :class="$style.titleSection">
				<n8n-select v-if="parentNodes.length" :popper-append-to-body="true" size="small" :value="currentNodeName" @input="onSelect" :no-data-text="$locale.baseText('ndv.input.noNodesFound')" :placeholder="$locale.baseText('ndv.input.parentNodes')" filterable>
					<template slot="prepend">
						<span :class="$style.title">{{ $locale.baseText('ndv.input') }}</span>
					</template>
					<n8n-option v-for="node in parentNodes" :value="node.name" :key="node.name" class="node-option">
						{{ truncate(node.name) }}&nbsp;
						<span >{{ $locale.baseText('ndv.input.nodeDistance', {adjustToNumber: node.depth}) }}</span>
					</n8n-option>
				</n8n-select>
				<span v-else :class="$style.title">{{ $locale.baseText('ndv.input') }}</span>
			</div>
		</template>

		<template v-slot:node-not-run>
			<div :class="$style.noOutputData" v-if="parentNodes.length">
				<n8n-text tag="div" :bold="true" color="text-dark" size="large">{{ $locale.baseText('ndv.input.noOutputData.title') }}</n8n-text>
				<NodeExecuteButton type="outline" :transparent="true" :nodeName="currentNodeName" :label="$locale.baseText('ndv.input.noOutputData.executePrevious')" @execute="onNodeExecute" />
				<n8n-text tag="div" size="small">
					{{ $locale.baseText('ndv.input.noOutputData.hint') }}
				</n8n-text>
			</div>
			<div :class="$style.notConnected" v-else>
				<div>
					<WireMeUp />
				</div>
				<n8n-text tag="div" :bold="true" color="text-dark" size="large">{{ $locale.baseText('ndv.input.notConnected.title') }}</n8n-text>
				<n8n-text tag="div">
					{{ $locale.baseText('ndv.input.notConnected.message') }}
					<a href="https://docs.n8n.io/workflows/connections/" target="_blank" @click="onConnectionHelpClick">
						{{$locale.baseText('ndv.input.notConnected.learnMore')}}
					</a>
				</n8n-text>
			</div>
		</template>

		<template v-slot:no-output-data>
			<n8n-text tag="div" :bold="true" color="text-dark" size="large">{{ $locale.baseText('ndv.input.noOutputData') }}</n8n-text>
		</template>
	</RunData>
</template>

<script lang="ts">
import { INodeUi } from '@/Interface';
import { IConnectedNode, Workflow } from 'n8n-workflow';
import RunData from './RunData.vue';
import { workflowHelpers } from '@/components/mixins/workflowHelpers';
import mixins from 'vue-typed-mixins';
import NodeExecuteButton from './NodeExecuteButton.vue';
import WireMeUp from './WireMeUp.vue';

export default mixins(
	workflowHelpers,
).extend({
	name: 'InputPanel',
	components: { RunData, NodeExecuteButton, WireMeUp },
	props: {
		currentNodeName: {
			type: String,
		},
		runIndex: {
			type: Number,
		},
		linkedRuns: {
			type: Boolean,
		},
		workflow: {
		},
		canLinkRuns: {
			type: Boolean,
		},
		sessionId: {
			type: String,
		},
	},
	computed: {
		isExecutingPrevious(): boolean {
			if (!this.workflowRunning) {
				return false;
			}
			const triggeredNode = this.$store.getters.executedNode;
			const executingNode = this.$store.getters.executingNode;
			if (this.activeNode && triggeredNode === this.activeNode.name && this.activeNode.name !== executingNode) {
				return true;
			}

			if (executingNode || triggeredNode) {
				return !!this.parentNodes.find((node) => node.name === executingNode || node.name === triggeredNode);
			}
			return false;
		},
		workflowRunning (): boolean {
			return this.$store.getters.isActionActive('workflowRunning');
		},
		currentWorkflow(): Workflow {
			return this.workflow as Workflow;
		},
		activeNode (): INodeUi | null {
			return this.$store.getters.activeNode;
		},
		currentNode (): INodeUi {
			return this.$store.getters.getNodeByName(this.currentNodeName);
		},
		connectedCurrentNodeOutputs(): number[] | undefined {
			const search = this.parentNodes.find(({name}) => name === this.currentNodeName);
			if (search) {
				return search.indicies;
			}
			return undefined;
		},
		parentNodes (): IConnectedNode[] {
			if (!this.activeNode) {
				return [];
			}
			const nodes: IConnectedNode[] = (this.workflow as Workflow).getParentNodesByDepth(this.activeNode.name);

			return nodes.filter(({name}, i) => (this.activeNode && (name !== this.activeNode.name)) && nodes.findIndex((node) => node.name === name) === i);
		},
	},
	methods: {
		onNodeExecute() {
			this.$emit('execute');
			if (this.activeNode) {
				this.$telemetry.track('User clicked ndv button', {
					node_type: this.activeNode.type,
					workflow_id: this.$store.getters.workflowId,
					session_id: this.sessionId,
					pane: 'input',
					type: 'executePrevious',
				});
			}
		},
		onRunIndexChange(run: number) {
			this.$emit('runChange', run);
		},
		onLinkRun() {
			this.$emit('linkRun');
		},
		onUnlinkRun() {
			this.$emit('unlinkRun');
		},
		onSelect(value: string) {
			const index = this.parentNodes.findIndex((node) => node.name === value) + 1;
			this.$emit('select', value, index);
		},
		onConnectionHelpClick() {
			if (this.activeNode) {
				this.$telemetry.track('User clicked ndv link', {
					node_type: this.activeNode.type,
					workflow_id: this.$store.getters.workflowId,
					session_id: this.sessionId,
					pane: 'input',
					type: 'not-connected-help',
				});
			}
		},
		truncate(nodeName: string) {
			const truncated = nodeName.substring(0, 30);
			if (truncated.length < nodeName.length) {
				return `${truncated}...`;
			}
			return truncated;
		},
	},
});
</script>

<style lang="scss" module>
.titleSection {
	display: flex;
	max-width: 300px;

	> * {
		margin-right: var(--spacing-2xs);
	}
}

.noOutputData {
	max-width: 180px;

	> *:first-child {
		margin-bottom: var(--spacing-m);
	}

	> * {
		margin-bottom: var(--spacing-2xs);
	}
}

.notConnected {
	max-width: 300px;

	> *:first-child {
		margin-bottom: var(--spacing-m);
	}

	> * {
		margin-bottom: var(--spacing-2xs);
	}
}

.title {
	text-transform: uppercase;
	color: var(--color-text-light);
	letter-spacing: 3px;
	font-size: var(--font-size-s);
	font-weight: var(--font-weight-bold);
}
</style>

<style lang="scss" scoped>
.node-option {
	font-weight: var(--font-weight-regular) !important;

	span {
		color: var(--color-text-light);
	}

	&.selected > span {
		color: var(--color-primary);
	}
}
</style>
