<template>
	<div :class="$style.container">
		<div
			:class="$style.connections"
			:style="`--possible-connections: ${possibleConnections.length}`"
		>
			<div v-for="connection in possibleConnections" :key="connection.type">
				<n8n-tooltip
					placement="top"
					:teleported="true"
					:offset="10"
					:disabled="
						!shouldShowConnectionTooltip(connection.type) ||
						connectedNodes[connection.type].length === 0
					"
				>
					<template #content>
						{{ connection.displayName }}
					</template>
					<div :class="$style.connectionType">
						<span :class="$style.connectionLabel" v-text="connection.displayName" />
						<div
							v-on-click-outside="() => expandConnectionGroup(connection.type, false)"
							:class="{
								[$style.connectedNodesWrapper]: true,
								[$style.connectedNodesExpanded]: expandedGroups.includes(connection.type),
							}"
							:style="`--nodes-length: ${connectedNodes[connection.type].length}`"
							@click="expandConnectionGroup(connection.type, true)"
						>
							<div
								v-if="connectedNodes[connection.type].length > 0"
								:class="{
									[$style.connectedNodes]: true,
									[$style.connectedNodesMultiple]: connectedNodes[connection.type].length > 1,
								}"
							>
								<div
									v-for="(node, index) in connectedNodes[connection.type]"
									:key="node.node.name"
									:class="{ [$style.nodeWrapper]: true, [$style.hasIssues]: node.issues }"
									:style="`--node-index: ${index}`"
								>
									<n8n-tooltip
										:key="node.node.name"
										placement="top"
										:teleported="true"
										:offset="10"
										:disabled="shouldShowConnectionTooltip(connection.type)"
									>
										<template #content>
											{{ node.node.name }}
											<template v-if="node.issues">
												<TitledList
													:title="`${$locale.baseText('node.issues')}:`"
													:items="node.issues"
												/>
											</template>
										</template>

										<div
											:class="$style.connectedNode"
											data-test-id="floating-node"
											:data-node-name="node.node.name"
											@click="onNodeClick(node.node.name, connection.type)"
										>
											<NodeIcon
												:node-type="node.nodeType"
												:node-name="node.node.name"
												tooltip-position="top"
												:size="20"
												circle
											/>
										</div>
									</n8n-tooltip>
								</div>
							</div>
							<div
								v-if="
									connectedNodes[connection.type].length >= 1
										? connection.maxConnections !== 1
										: true
								"
								:class="$style.plusButton"
								@click="onPlusClick(connection.type)"
							>
								<n8n-tooltip
									placement="top"
									:teleported="true"
									:offset="10"
									:disabled="
										shouldShowConnectionTooltip(connection.type) &&
										connectedNodes[connection.type].length >= 1
									"
								>
									<template #content> Add {{ connection.displayName }} </template>
									<n8n-icon-button size="medium" icon="plus" type="tertiary" />
								</n8n-tooltip>
							</div>
						</div>
					</div>
				</n8n-tooltip>
			</div>
		</div>
	</div>
</template>

<script setup lang="ts">
import type { INodeUi } from '@/Interface';
import { useNodeTypesStore } from '@/stores/nodeTypes.store';
import { useWorkflowsStore } from '@/stores/workflows.store';
import { computed, ref, watch } from 'vue';
import { NodeHelpers } from 'n8n-workflow';
import { useNodeHelpers } from '@/composables/useNodeHelpers';
import NodeIcon from '@/components/NodeIcon.vue';
import TitledList from '@/components/TitledList.vue';
import type { ConnectionTypes, INodeInputConfiguration, INodeTypeDescription } from 'n8n-workflow';

interface Props {
	rootNode: INodeUi;
}

const props = defineProps<Props>();
const workflowsStore = useWorkflowsStore();
const nodeTypesStore = useNodeTypesStore();
const nodeHelpers = useNodeHelpers();
const emit = defineEmits(['switchSelectedNode', 'openConnectionNodeCreator']);

interface NodeConfig {
	node: INodeUi;
	nodeType: INodeTypeDescription;
	issues: string[];
}

const possibleConnections = ref<INodeInputConfiguration[]>([]);

const expandedGroups = ref<ConnectionTypes[]>([]);

const nodeType = computed(() =>
	nodeTypesStore.getNodeType(props.rootNode.type, props.rootNode.typeVersion),
);

const nodeData = computed(() => workflowsStore.getNodeByName(props.rootNode.name));

const workflow = computed(() => workflowsStore.getCurrentWorkflow());

function getConnectionConfig(connectionType: ConnectionTypes) {
	return possibleConnections.value.find((c) => c.type === connectionType);
}

function isConnectionExpandable(connectionType: ConnectionTypes) {
	const connectionConfig = getConnectionConfig(connectionType);
	return connectionConfig?.maxConnections !== 1;
}

function shouldShowConnectionTooltip(connectionType: ConnectionTypes) {
	return isConnectionExpandable(connectionType) && !expandedGroups.value.includes(connectionType);
}

function expandConnectionGroup(connectionType: ConnectionTypes, isExpanded: boolean) {
	// If the connection is a single connection, we don't need to expand the group
	if (!isConnectionExpandable(connectionType)) {
		return;
	}

	if (isExpanded) {
		expandedGroups.value = [...expandedGroups.value, connectionType];
	} else {
		expandedGroups.value = expandedGroups.value.filter((g) => g !== connectionType);
	}
}

function getINodesFromNames(names: string[]): NodeConfig[] {
	return names
		.map((name) => {
			const node = workflowsStore.getNodeByName(name);
			if (node) {
				const matchedNodeType = nodeTypesStore.getNodeType(node.type);
				if (matchedNodeType) {
					const issues = nodeHelpers.getNodeIssues(matchedNodeType, node, workflow.value);
					const stringifiedIssues = issues ? NodeHelpers.nodeIssuesToString(issues, node) : '';
					return { node, nodeType: matchedNodeType, issues: stringifiedIssues };
				}
			}
			return null;
		})
		.filter((n): n is NodeConfig => n !== null);
}

const connectedNodes = computed<Record<ConnectionTypes, NodeConfig[]>>(() => {
	return possibleConnections.value.reduce(
		(acc, connection) => {
			const nodes = getINodesFromNames(
				workflow.value.getParentNodes(props.rootNode.name, connection.type),
			);
			return { ...acc, [connection.type]: nodes };
		},
		{} as Record<ConnectionTypes, NodeConfig[]>,
	);
});

function isNodeInputConfiguration(
	connectionConfig: ConnectionTypes | INodeInputConfiguration,
): connectionConfig is INodeInputConfiguration {
	if (typeof connectionConfig === 'string') return false;

	return 'type' in connectionConfig;
}

function getPossibleSubInputConnections(): INodeInputConfiguration[] {
	if (!nodeType.value || !props.rootNode) return [];

	const inputs = NodeHelpers.getNodeInputs(workflow.value, props.rootNode, nodeType.value);

	const nonMainInputs = inputs.filter((input): input is INodeInputConfiguration => {
		if (!isNodeInputConfiguration(input)) return false;

		return input.type !== 'main';
	});

	return nonMainInputs;
}

function onNodeClick(nodeName: string, connectionType: ConnectionTypes) {
	if (isConnectionExpandable(connectionType) && !expandedGroups.value.includes(connectionType)) {
		expandConnectionGroup(connectionType, true);
		return;
	}

	emit('switchSelectedNode', nodeName);
}

function onPlusClick(connectionType: ConnectionTypes) {
	const connectionNodes = connectedNodes.value[connectionType];
	if (
		isConnectionExpandable(connectionType) &&
		!expandedGroups.value.includes(connectionType) &&
		connectionNodes.length >= 1
	) {
		expandConnectionGroup(connectionType, true);
		return;
	}

	emit('openConnectionNodeCreator', props.rootNode.name, connectionType);
}

watch(
	nodeData,
	() =>
		setTimeout(() => {
			expandedGroups.value = [];
			possibleConnections.value = getPossibleSubInputConnections();
		}, 0),
	{ immediate: true },
);
</script>

<style lang="scss" module>
.container {
	--animation-duration: 200ms;
	padding-top: 50px;
}
.connections {
	z-index: 11;
	position: absolute;
	bottom: -20px;
	left: 0;
	right: 0;
	user-select: none;
	justify-content: space-between;
	display: grid;
	grid-template-columns: repeat(var(--possible-connections), 1fr);
}
.connectionType {
	display: flex;
	flex-direction: column;
	align-items: center;
	transition: all var(--animation-duration) ease;
}
.connectionLabel {
	margin-bottom: var(--spacing-2xs);
	font-size: var(--font-size-2xs);
	user-select: none;
	text-wrap: nowrap;
}
.connectedNodesWrapper {
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: space-between;
	position: relative;
}
.plusButton {
	transition: all var(--animation-duration) ease;
	position: absolute;
	top: var(--spacing-2xs);

	&:not(:first-child) {
		z-index: 0;
		left: 100%;
		margin-left: -15px;

		.connectedNodesExpanded & {
			margin-left: var(--spacing-2xs);
			opacity: 1;
		}
	}
}

.connectedNodesMultiple {
	transition: all var(--animation-duration) ease;
}
.connectedNodesExpanded {
	z-index: 10;
	filter: drop-shadow(0px 0px 10px rgba(0, 0, 0, 0.2));
}
// Hide all other connection groups when one is expanded
.connections:has(.connectedNodesExpanded) .connectionType:not(:has(.connectedNodesExpanded)) {
	opacity: 0;
	pointer-events: none;
}
.connectedNode {
	border: var(--border-base);
	background-color: var(--color-canvas-node-background);
	border-radius: 100%;
	padding: var(--spacing-xs);
	cursor: pointer;
	pointer-events: all;
	transition: all var(--animation-duration) ease;
	position: relative;
	display: flex;
	justify-self: center;
	align-self: center;
}
.connectedNodes {
	display: flex;
	justify-content: center;
	margin-right: calc((var(--nodes-length) - 1) * -35px);
	.connectedNodesExpanded & {
		margin-right: 0;
	}
}
.nodeWrapper {
	--collapsed-node-offset: calc(var(--node-index) * -35px);
	transition: all var(--animation-duration) ease;
	transform-origin: center;
	z-index: 10;
	.connectedNodesExpanded &:not(:first-child) {
		margin-left: var(--spacing-2xs);
	}
	&.hasIssues {
		.connectedNode {
			border-width: 2px;
			border-color: var(--color-danger);
		}
	}

	&:not(:first-child) {
		transform: translateX(var(--collapsed-node-offset));
	}

	.connectedNodesExpanded & {
		transform: translateX(0);
	}
}
</style>