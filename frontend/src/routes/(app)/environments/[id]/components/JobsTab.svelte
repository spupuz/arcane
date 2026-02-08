<script lang="ts">
	import { jobScheduleService } from '$lib/services/job-schedule-service';
	import { containerService } from '$lib/services/container-service';
	import { tryCatch } from '$lib/utils/try-catch';
	import JobCard from '$lib/components/job-card/job-card.svelte';
	import { Spinner } from '$lib/components/ui/spinner';
	import { m } from '$lib/paraglide/messages';
	import * as Card from '$lib/components/ui/card';
	import { Label } from '$lib/components/ui/label';
	import { Switch } from '$lib/components/ui/switch';
	import { JobsIcon, AlertIcon } from '$lib/icons';
	import type { JobStatus, JobPrerequisite } from '$lib/types/job-schedule.type';
	import type { ContainerSummaryDto } from '$lib/types/container.type';

	let { formInputs, environmentId } = $props();

	let refreshSignal = $state(0);

	const jobsPromise = $derived.by(async () => {
		refreshSignal; // trigger dependency
		if (!environmentId) return null;

		const result = await tryCatch(jobScheduleService.listJobs(environmentId));

		if (result.error) {
			throw result.error;
		}

		return {
			...result.data,
			jobs: result.data.jobs.map((job) => ({
				...job,
				prerequisites: job.prerequisites.map((prereq) => ({
					...prereq,
					settingsUrl: resolveSettingsUrl(job, prereq)
				}))
			}))
		};
	});

	const containersPromise = $derived.by(async () => {
		if (!$formInputs.autoUpdate.value) return [];
		const result = await tryCatch(containerService.getContainers({ pagination: { page: 1, limit: 100 } }));
		if (result.error) throw result.error;
		return result.data.data;
	});

	// Use $derived to compute the excluded containers set from form input (avoids $effect state mutation)
	const excludedContainers = $derived.by(() => {
		const savedValue = $formInputs.autoUpdateExcludedContainers?.value || '';
		const names = savedValue
			.split(',')
			.map((s: string) => normalizeContainerName(s.trim()))
			.filter(Boolean);
		return new Set<string>(names);
	});

	let containerSearchQuery = $state('');

	const exclusionLabel = $derived.by(() => {
		if (excludedContainers.size === 0) return m.auto_update_select_containers();
		if (excludedContainers.size === 1) return m.auto_update_containers_excluded_one();
		return m.auto_update_containers_excluded_many({ count: excludedContainers.size });
	});

	function resolveSettingsUrl(job: JobStatus, prereq: JobPrerequisite): string | undefined {
		if (!prereq.settingsUrl) return undefined;
		if (!environmentId) return prereq.settingsUrl;

		const envBase = `/environments/${environmentId}`;
		switch (prereq.settingKey) {
			case 'pollingEnabled':
			case 'autoUpdate':
				return `${envBase}?tab=docker`;
			case 'gitopsSyncEnabled':
				return `${envBase}?tab=gitops`;
			case 'scheduledPruneEnabled':
				return undefined;
			case 'vulnerabilityScanEnabled':
				return undefined;
			default:
				return prereq.settingsUrl;
		}
	}

	function loadJobs() {
		refreshSignal++;
	}

	function toggleContainerExclusion(containerName: string) {
		const normalizedName = normalizeContainerName(containerName);
		const currentExcluded = new Set(excludedContainers);
		
		if (currentExcluded.has(normalizedName)) {
			currentExcluded.delete(normalizedName);
		} else {
			currentExcluded.add(normalizedName);
		}

		const newValue = Array.from(currentExcluded).join(',');
		if ($formInputs.autoUpdateExcludedContainers) {
			$formInputs.autoUpdateExcludedContainers.value = newValue;
		}
	}

	const categories = [
		{ id: 'monitoring', label: m.jobs_monitoring_heading() },
		{ id: 'maintenance', label: m.jobs_maintenance_heading() },
		{ id: 'security', label: m.jobs_security_heading() },
		{ id: 'updates', label: m.jobs_updates_heading() },
		{ id: 'sync', label: m.jobs_sync_heading() },
		{ id: 'telemetry', label: m.jobs_telemetry_heading() }
	];

	const hiddenJobIds = new Set(['gitops-sync', 'filesystem-watcher']);

	function getJobsByCategory(categoryId: string, jobs: JobStatus[]): JobStatus[] {
		return jobs.filter((j) => {
			if (hiddenJobIds.has(j.id)) return false;
			if (j.category !== categoryId) return false;
			// Only show manager-only jobs on the local environment (ID "0")
			if (j.managerOnly && environmentId !== '0') return false;
			return true;
		});
	}

	function getEnabledOverride(job: JobStatus): boolean | undefined {
		switch (job.id) {
			case 'scheduled-prune':
				return $formInputs.scheduledPruneEnabled.value;
			case 'auto-update':
				return $formInputs.autoUpdate.value;
			case 'image-polling':
				return $formInputs.pollingEnabled.value;
			case 'vulnerability-scan':
				return $formInputs.vulnerabilityScanEnabled.value;
			default:
				return undefined;
		}
	}

	function getContainerName(c: ContainerSummaryDto): string {
		const rawName = c.names[0] || c.id.substring(0, 12);
		return normalizeContainerName(rawName);
	}

	function normalizeContainerName(name: string): string {
		return name.replace(/^\/+/, '');
	}

	function isContainerLabelExcluded(container: ContainerSummaryDto): boolean {
		const labels = container.labels || {};
		for (const [k, v] of Object.entries(labels)) {
			if (k.toLowerCase() === 'com.getarcaneapp.arcane.updater') {
				return ['false', '0', 'no', 'off'].includes(v.trim().toLowerCase());
			}
		}
		return false;
	}

	function mapContainerToItem(container: ContainerSummaryDto) {
		const name = getContainerName(container);
		const labelExcluded = isContainerLabelExcluded(container);
		return {
			value: name,
			label: name,
			disabled: labelExcluded,
			hint: labelExcluded ? '(Label)' : undefined,
			selected: excludedContainers.has(name) || labelExcluded
		};
	}
</script>

<div class="space-y-6">
	<Card.Root>
		<Card.Header icon={JobsIcon}>
			<div class="flex flex-col space-y-1.5">
				<Card.Title>
					<h2>{m.jobs_title()}</h2>
				</Card.Title>
				<Card.Description>{m.jobs_environment_scope_description()}</Card.Description>
			</div>
		</Card.Header>
		<Card.Content class="p-4 sm:p-6">
			{#await jobsPromise}
				<div class="flex h-32 items-center justify-center">
					<Spinner class="size-8" />
				</div>
			{:then jobsResponse}
				{#if jobsResponse}
					<div class="space-y-8">
						{#each categories as category (category.id)}
							{@const categoryJobs = getJobsByCategory(category.id, jobsResponse.jobs)}
							{#if categoryJobs.length > 0}
								<div class="space-y-4">
									<h3 class="text-muted-foreground text-sm font-semibold tracking-tight uppercase">
										{category.label}
									</h3>
									<div class="grid gap-4 sm:grid-cols-2 lg:grid-cols-2">
										{#each categoryJobs as job (job.id)}
											<JobCard
												{job}
												{environmentId}
												isAgent={jobsResponse.isAgent}
												onScheduleUpdate={loadJobs}
												enabledOverride={getEnabledOverride(job)}
											>
												{#snippet headerAccessory()}
													{#if job.id === 'image-polling'}
														<Switch bind:checked={$formInputs.pollingEnabled.value} />
													{:else if job.id === 'auto-update'}
														<Switch bind:checked={$formInputs.autoUpdate.value} disabled={!$formInputs.pollingEnabled.value} />
													{:else if job.id === 'scheduled-prune'}
														<Switch bind:checked={$formInputs.scheduledPruneEnabled.value} />
													{:else if job.id === 'vulnerability-scan'}
														<Switch bind:checked={$formInputs.vulnerabilityScanEnabled.value} />
													{/if}
												{/snippet}

												{#if job.id === 'auto-update' && $formInputs.autoUpdate.value}
													<div class="border-border/20 space-y-3 border-t pt-3">
														<div class="space-y-1">
															<Label class="text-sm font-medium">Excluded Containers</Label>
									{#if excludedContainers.size > 0}
										<span class="bg-primary/10 text-primary ml-2 rounded-full px-2 py-0.5 text-xs font-medium">{excludedContainers.size} excluded</span>
									{/if}
															<p class="text-muted-foreground text-xs">Select containers to exclude from automatic updates.</p>
														</div>

														{#await containersPromise}
															<div class="border-input bg-background rounded-md border p-2">
																<div class="flex items-center justify-center py-4">
																	<Spinner class="size-4" />
																	<span class="text-muted-foreground ml-2 text-sm">Loading containers...</span>
																</div>
															</div>
														{:then containers}
															{@const allContainerItems = containers.map(mapContainerToItem)}
															{@const filteredContainerItems = containerSearchQuery
																? allContainerItems.filter((item) => item.label.toLowerCase().includes(containerSearchQuery.toLowerCase()))
																: allContainerItems}
															<div class="border-input bg-background rounded-md border">
																<div class="border-b p-2">
																	<input
																		type="text"
																		placeholder="Search containers..."
																		bind:value={containerSearchQuery}
																		class="bg-transparent text-sm outline-none placeholder:text-muted-foreground w-full"
																	/>
																</div>
																<div class="max-h-48 overflow-y-auto p-1">
																	{#if filteredContainerItems.length === 0}
																		<div class="text-muted-foreground py-2 text-center text-sm">No containers found</div>
																	{:else}
																		{#each filteredContainerItems as item (item.value)}
																			<button
																				type="button"
																				class="hover:bg-accent flex w-full items-center gap-2 rounded-sm px-2 py-1.5 text-sm transition-colors"
																				class:opacity-50={item.disabled}
																				disabled={item.disabled}
																				onclick={() => toggleContainerExclusion(item.value)}
																			>
																				<div
																					class="border-primary flex size-4 shrink-0 items-center justify-center rounded-sm border {item.selected
																						? 'bg-primary text-primary-foreground'
																						: 'opacity-50'}"
																				>
																					{#if item.selected}
																						<svg class="size-3" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="3">
																							<polyline points="20 6 9 17 4 12"></polyline>
																						</svg>
																					{/if}
																				</div>
																				<span class="truncate" class:text-muted-foreground={item.disabled}>{item.label}</span>
																				{#if item.hint}
																					<span class="text-muted-foreground ml-auto text-xs">{item.hint}</span>
																				{/if}
																			</button>
																		{/each}
																	{/if}
																</div>
															</div>
														{:catch error}
															<div class="border-destructive/50 bg-destructive/10 text-destructive rounded-md border p-3 text-sm">
																{error.message || 'Failed to load containers'}
															</div>
														{/await}
													</div>
												{/if}

												{#if job.id === 'scheduled-prune'}
													{#if $formInputs.scheduledPruneEnabled.value}
														<div class="border-border/20 space-y-4 border-t pt-3">
															<div class="grid gap-3 sm:grid-cols-2">
																<div class="bg-muted/20 ring-border/20 flex items-start justify-between rounded-lg p-3 ring-1">
																	<div class="space-y-0.5">
																		<Label class="text-sm font-medium">{m.scheduled_prune_containers_label()}</Label>
																		<p class="text-muted-foreground text-xs">{m.scheduled_prune_containers_description()}</p>
																	</div>
																	<Switch bind:checked={$formInputs.scheduledPruneContainers.value} />
																</div>
																<div class="bg-muted/20 ring-border/20 flex items-start justify-between rounded-lg p-3 ring-1">
																	<div class="space-y-0.5">
																		<Label class="text-sm font-medium">{m.scheduled_prune_images_label()}</Label>
																		<p class="text-muted-foreground text-xs">{m.scheduled_prune_images_description()}</p>
																	</div>
																	<Switch bind:checked={$formInputs.scheduledPruneImages.value} />
																</div>
																<div class="bg-muted/20 ring-border/20 flex items-start justify-between rounded-lg p-3 ring-1">
																	<div class="space-y-0.5">
																		<Label class="text-sm font-medium">{m.scheduled_prune_volumes_label()}</Label>
																		<p class="text-muted-foreground text-xs">{m.scheduled_prune_volumes_description()}</p>
																	</div>
																	<Switch bind:checked={$formInputs.scheduledPruneVolumes.value} />
																</div>
																<div class="bg-muted/20 ring-border/20 flex items-start justify-between rounded-lg p-3 ring-1">
																	<div class="space-y-0.5">
																		<Label class="text-sm font-medium">{m.scheduled_prune_networks_label()}</Label>
																		<p class="text-muted-foreground text-xs">{m.scheduled_prune_networks_description()}</p>
																	</div>
																	<Switch bind:checked={$formInputs.scheduledPruneNetworks.value} />
																</div>
																<div class="bg-muted/20 ring-border/20 flex items-start justify-between rounded-lg p-3 ring-1">
																	<div class="space-y-0.5">
																		<Label class="text-sm font-medium">{m.scheduled_prune_build_cache_label()}</Label>
																		<p class="text-muted-foreground text-xs">{m.scheduled_prune_build_cache_description()}</p>
																	</div>
																	<Switch bind:checked={$formInputs.scheduledPruneBuildCache.value} />
																</div>
															</div>
															{#if $formInputs.scheduledPruneVolumes.value}
																<div
																	class="flex items-start gap-3 rounded-lg border border-amber-500/30 bg-amber-500/10 p-3 text-amber-900 dark:text-amber-200"
																>
																	<AlertIcon class="mt-0.5 size-4 shrink-0 text-amber-600 dark:text-amber-400" />
																	<div class="space-y-1 text-sm">
																		<p class="font-medium">{m.scheduled_prune_volumes_warning()}</p>
																	</div>
																</div>
															{/if}
														</div>
													{/if}
												{/if}
											</JobCard>
										{/each}
									</div>
								</div>
							{/if}
						{/each}
					</div>
				{/if}
			{:catch error}
				<div class="border-destructive/50 bg-destructive/10 text-destructive rounded-lg border p-4">
					{error.message || error}
				</div>
			{/await}
		</Card.Content>
	</Card.Root>
</div>