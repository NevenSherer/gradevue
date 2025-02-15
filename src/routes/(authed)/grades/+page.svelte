<script lang="ts">
	import { getColorForGrade, removeClassID } from '$lib';
	import { gradebook, gradebookLoaded, studentAccount } from '$lib/stores';
	import { Button, Card, Dropdown, DropdownItem, Progressbar } from 'flowbite-svelte';
	import { ChevronDownOutline, ChevronUpOutline, MapPinAltOutline } from 'flowbite-svelte-icons';

	let dropdownOpen = $state(false);


	const changeReportPeriod = async (period: number) => {
		dropdownOpen = false;

		$gradebookLoaded = false;
		$gradebook = await $studentAccount?.grades(period);
		$gradebookLoaded = true;
	};
</script>

<svelte:head>
	<title>Grades - GradeVue</title>
</svelte:head>

{#if $gradebook}
	<div class="m-4 flex flex-col justify-center">
		<Button color="light" class="mx-auto flex items-center">
			{$gradebook.ReportingPeriod._GradePeriod}

			{#if dropdownOpen}
				<ChevronUpOutline size="sm" class="ml-2 focus:outline-none" />
			{:else}
				<ChevronDownOutline size="sm" class="ml-2 focus:outline-none" />
			{/if}
		</Button>

		<Dropdown bind:open={dropdownOpen}>
			{#each $gradebook.ReportingPeriods.ReportPeriod ?? [] as period, index}
				<DropdownItem onclick={() => changeReportPeriod(index)} class="flex items-center">
					{#if period._GradePeriod == $gradebook.ReportingPeriod._GradePeriod}
						<MapPinAltOutline size="sm" class="mr-2" />
					{/if}
					{period._GradePeriod}
				</DropdownItem>
			{/each}
		</Dropdown>
	</div>

	<ol class="space-y-4 p-4 pt-0">
		{#each $gradebook.Courses.Course ?? [] as { _Title: title, Marks: { Mark: { _CalculatedScoreString: grade, _CalculatedScoreRaw: percent } } }, index}
			<li>
				<Card
					class="dark:text-white text-xl max-w-none flex flex-row justify-between items-center"
					href="/grades/{index.toString()}"
				>
					<span class="line-clamp-1 mr-2">{removeClassID(title)}</span>
					<span class="shrink-0 ml-auto mr-2">
						{grade}
						{parseFloat(percent)}%
					</span>

					<Progressbar
						color={getColorForGrade(grade)}
						progress={Math.min(isNaN(parseFloat(percent)) ? 0 : parseFloat(percent), 100)}
						animate={true}
						class="hidden sm:block w-1/3 shrink-0"
					/>
				</Card>
			</li>
		{/each}
	</ol>
{/if}
