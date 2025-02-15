<script lang="ts">
	import { getColorForGrade } from '$lib';
	import { calculateGradePercentage } from '$lib/assignments';
	import DateBadge from '$lib/components/DateBadge.svelte';
	import {
		Badge,
		Button,
		Card,
		Checkbox,
		Dropdown,
		DropdownItem,
		Input,
		NumberInput,
		Popover,
		Progressbar
	} from 'flowbite-svelte';
	import { ChevronDownOutline, InfoCircleOutline } from 'flowbite-svelte-icons';

	interface Props {
		name: string;
		pointsEarned?: number;
		pointsPossible?: number;
		extraCredit?: boolean;
		gradePercentageChange?: number;
		notForGrade?: boolean;
		hidden?: boolean;
		showHypotheticalLabel?: boolean;
		category?: string | undefined;
		categoryDropdownOptions?: string[];
		date?: Date | undefined;
		editable?: boolean;
		recalculateGradePercentage?: () => void;
	}

	let {
		name = $bindable(),
		pointsEarned = $bindable(),
		pointsPossible = $bindable(),
		extraCredit = $bindable(false),
		gradePercentageChange,
		notForGrade = $bindable(false),
		hidden = false,
		showHypotheticalLabel = false,
		category = $bindable(undefined),
		categoryDropdownOptions = [],
		date = undefined,
		editable = false,
		recalculateGradePercentage = () => {}
	}: Props = $props();

	let categoryDropdownOpen = $state(false);

	const getCategoryColor = (category: string) => {
		if (category.match(/final/i)) return 'red';
		if (category.match(/test|quiz|assessment|performance/i)) return 'purple';
		if (category.match(/homework|classwork|activity|activities|assignment|project/i))
			return 'green';
		return 'primary';
	};

	let percentage = $derived(
		pointsEarned && pointsPossible ? calculateGradePercentage(pointsEarned, pointsPossible) : 0
	);

	let percentageChange = $derived(Math.round((gradePercentageChange ?? 0) * 100) / 100);
</script>

<Card class="dark:text-white max-w-none flex flex-row items-center sm:p-4">
	<div class="mr-2">
		{#if editable}
			<Input bind:value={name} class="w-48 inline" />

			{#if categoryDropdownOptions.length > 0}
				<Button color="light">
					{category ?? 'Category'}
					<ChevronDownOutline size="xs" class="ml-2 focus:outline-none" />
				</Button>

				<Dropdown bind:open={categoryDropdownOpen}>
					{#each categoryDropdownOptions as categoryOption}
						<DropdownItem
							onclick={() => {
								category = categoryOption;
								categoryDropdownOpen = false;
								recalculateGradePercentage();
							}}
						>
							{categoryOption}
						</DropdownItem>
					{/each}
				</Dropdown>
			{/if}
		{:else}
			<span>{name}</span>
		{/if}
		{#if category && (!editable || categoryDropdownOptions.length === 0) && !showHypotheticalLabel}
			<Badge color={getCategoryColor(category)}>
				{category}
			</Badge>
		{/if}
		{#if extraCredit}
			<Badge border color="indigo">
				{#if editable}
					<Checkbox bind:checked={extraCredit} onchange={recalculateGradePercentage}>
						<span class="text-xs">Extra Credit</span>
					</Checkbox>
				{:else}
					Extra Credit
				{/if}
			</Badge>
		{/if}
		{#if pointsEarned === undefined}
			<Badge border color="purple">Not Graded</Badge>
		{/if}
		{#if notForGrade}
			<Badge border color="pink">
				{#if editable}
					<Checkbox bind:checked={notForGrade} onchange={recalculateGradePercentage}>
						<span class="text-xs">Not For Grade</span>
					</Checkbox>
				{:else}
					Not For Grade
				{/if}
			</Badge>
		{/if}
		{#if hidden}
			<Popover triggeredBy=".hidden-badge" class="max-w-md">
				Teachers can choose to have assignments hidden from the assignment list but still calculated
				toward your grade. GradeVue can reveal these assignments.
			</Popover>
			<Badge border color="dark" class="hidden-badge">
				Hidden Assignments <InfoCircleOutline size="xs" class="ml-1 focus:outline-none" />
			</Badge>
		{/if}
		{#if showHypotheticalLabel}
			<Badge border color="dark">Hypothetical Assignment</Badge>
		{/if}
		{#if date}
			<DateBadge {date} />
		{/if}
	</div>

	<div class="ml-auto mr-2 shrink-0 flex items-center gap-2">
		{#if gradePercentageChange !== undefined}
			{#if percentageChange < 0}
				<span class="text-red-500">
					{percentageChange}%
				</span>
			{:else if percentageChange > 0}
				<span class="text-green-500">
					+{percentageChange}%
				</span>
			{:else if !notForGrade && pointsEarned && !isNaN(pointsEarned)}
				<span class="text-gray-500">+0%</span>
			{/if}
		{/if}

		{#if editable}
			<div class="w-32 flex items-center">
				<NumberInput
					type="number"
					size="sm"
					bind:value={pointsEarned}
					oninput={recalculateGradePercentage}
				/>
				<span class="mx-1"> / </span>
				<NumberInput
					type="number"
					size="sm"
					bind:value={pointsPossible}
					oninput={recalculateGradePercentage}
				/>
			</div>
		{:else if pointsEarned === undefined}
			{pointsPossible}
		{:else}
			{pointsEarned}/{pointsPossible}
			{#if percentage && percentage != Infinity}
				{Math.round(percentage * 100) / 100}%
			{/if}
		{/if}
	</div>

	{#if pointsEarned !== undefined || editable}
		<Progressbar
			color={extraCredit ? 'blue' : getColorForGrade(percentage)}
			progress={Math.min(isNaN(percentage) ? 0 : percentage, 100)}
			animate={true}
			class="hidden sm:block w-1/3 shrink-0"
		/>
	{/if}
</Card>
