<script lang="ts">
	import { page } from '$app/stores';
	import { removeClassID } from '$lib';
	import {
		calculateAssignmentGPCs,
		calculateAssignmentGPCsFromCategories,
		calculateAssignmentGPCsFromTotals,
		calculateCourseGradePercentageFromCategories,
		calculateCourseGradePercentageFromTotals,
		calculateGradePercentage,
		type Category,
		type Flowed,
		getCalculableAssignments,
		getHiddenAssignmentsFromCategories,
		getPointsByCategory,
		getSynergyCourseAssignmentCategories,
		gradesMatch,
		type HiddenAssignment,
		type NewHypotheticalAssignment,
		type ReactiveAssignment,
		type RealAssignment
	} from '$lib/assignments';
	import { gradebook } from '$lib/stores';
	import {
		Alert,
		Button,
		Checkbox,
		Popover,
		TabItem,
		Table,
		TableBody,
		TableBodyCell,
		TableBodyRow,
		TableHead,
		TableHeadCell,
		Tabs
	} from 'flowbite-svelte';
	import {
		ChevronDownOutline,
		ChevronUpOutline,
		ExclamationCircleSolid,
		GridPlusOutline,
		InfoCircleOutline
	} from 'flowbite-svelte-icons';
	import { untrack } from 'svelte';
	import { fade } from 'svelte/transition';
	import AssignmentCard from './AssignmentCard.svelte';

	const synergyCourse = $derived($gradebook?.Courses.Course?.[parseInt($page.params.index)]);

	const courseName = $derived(synergyCourse ? removeClassID(synergyCourse._Title) : '');

	const synergyGradePercentage = $derived(
		parseFloat(synergyCourse?.Marks.Mark._CalculatedScoreRaw ?? '')
	);

	const categories: Category[] | undefined = $derived(
		synergyCourse ? getSynergyCourseAssignmentCategories(synergyCourse) : undefined
	);

	const gradeCategories = $derived(categories?.filter((category) => category.name !== 'TOTAL'));

	const totalCategory = $derived(categories?.find((category) => category.name === 'TOTAL'));

	const synergyAssignments = $derived(synergyCourse?.Marks.Mark.Assignments.Assignment ?? []);

	const realAssignments = $derived(
		calculateAssignmentGPCs(
			synergyAssignments.map((synergyAssignment) => {
				// Edge Cases:

				// Normal:
				// _Point: "3"
				// _PointPossible: "4"
				// _Points: "3 / 4"
				// _ScoreCalValue: "3"
				// _ScoreMaxValue: "4"
				// _DisplayScore: "3 out of 4"

				// Not Graded:
				// _Point: undefined
				// _PointPossible: undefined
				// _Points: "4 Points Possible"
				// _ScoreCalValue: undefined
				// _ScoreMaxValue: "4" or undefined
				// _DisplayScore: "Not Graded"

				// Extra Credit:
				// _Point: "3"
				// _PointPossible: ""
				// _Points: "3 /"
				// _ScoreCalValue: "3"
				// _ScoreMaxValue: "4"
				// _DisplayScore: "3 out of 4"

				// Not For Grading:
				// _Point: "3"
				// _PointPossible: "4"
				// _Points: "3 / 4"
				// _ScoreCalValue: "3"
				// _ScoreMaxValue: "4"
				// _DisplayScore: "3 out of 4"
				// _Notes : "(Not For Grading)"

				const pointsEarned = synergyAssignment._ScoreCalValue
					? parseFloat(synergyAssignment._ScoreCalValue)
					: undefined;

				const pointsPossible = synergyAssignment._ScoreMaxValue
					? parseFloat(synergyAssignment._ScoreMaxValue)
					: parseFloat(synergyAssignment._Points.split(' Points Possible')[0]); // Sometimes ScoreMaxValue is undefined; you can still get the points possible from the _Points field

				const assignment: RealAssignment = {
					name: synergyAssignment._Measure,
					pointsEarned,
					pointsPossible,
					extraCredit: synergyAssignment._PointPossible === '',
					gradePercentageChange: 0,
					notForGrade: synergyAssignment._Notes.includes('(Not For Grading)'),
					hidden: false,
					category: synergyAssignment._Type,
					date: new Date(synergyAssignment._Date),
					newHypothetical: false
				};

				return assignment;
			}),
			gradeCategories
		)
	);

	const hiddenAssignments = $derived(
		categories
			? getHiddenAssignmentsFromCategories(
					categories,
					getPointsByCategory(getCalculableAssignments(realAssignments))
				)
			: []
	);

	const assignments: (RealAssignment | Flowed<RealAssignment | HiddenAssignment>)[] = $derived([
		...hiddenAssignments,
		...realAssignments
	]);

	let hypotheticalMode = $state(false);

	let calculatedGrade = $state(NaN);
	let hypotheticalGrade = $state(NaN);
	let rawGradeCalcMatches = $derived(gradesMatch(calculatedGrade, synergyGradePercentage));

	let reactiveAssignments: ReactiveAssignment[] = $state([]);

	// Initialize reactive assignments and re-initialize them when assignments change
	$effect(() => {
		assignments;

		untrack(() => {
			reactiveAssignments = assignments.map((assignment) => {
				const reactiveAssignment: ReactiveAssignment = $state({
					...assignment,
					pointsEarned: assignment.pointsEarned,
					gradePercentageChange: assignment.gradePercentageChange,
					newHypothetical: false,
					reactive: true
				});

				return reactiveAssignment;
			});

			const calculable = getCalculableAssignments(assignments);

			calculatedGrade = gradeCategories
				? calculateCourseGradePercentageFromCategories(
						getPointsByCategory(calculable),
						gradeCategories
					)
				: calculateCourseGradePercentageFromTotals(calculable);

			hypotheticalGrade = calculatedGrade;
		});
	});

	function recalculateGradePercentage() {
		if (gradeCategories === undefined) {
			reactiveAssignments = calculateAssignmentGPCsFromTotals(reactiveAssignments);
		} else {
			reactiveAssignments = calculateAssignmentGPCsFromCategories(
				reactiveAssignments,
				gradeCategories
			);
		}
		const calculable = getCalculableAssignments(reactiveAssignments);

		hypotheticalGrade = gradeCategories
			? calculateCourseGradePercentageFromCategories(
					getPointsByCategory(calculable),
					gradeCategories
				)
			: calculateCourseGradePercentageFromTotals(calculable);
	}

	function addHypotheticalAssignment() {
		const newHypotheticalAssignment: NewHypotheticalAssignment = $state({
			name: 'Hypothetical Assignment',
			pointsEarned: undefined,
			pointsPossible: undefined,
			extraCredit: false,
			gradePercentageChange: undefined,
			notForGrade: false,
			hidden: false,
			category: 'Select category',
			newHypothetical: true,
			date: undefined,
			reactive: true
		});

		reactiveAssignments = [newHypotheticalAssignment, ...reactiveAssignments];
	}

	let calcWarningOpen = $state(false);
</script>

<svelte:head>
	<title>{courseName} - GradeVue</title>
</svelte:head>

{#if synergyCourse}
	<div class="h-12 md:h-14"></div>

	{#if categories && gradeCategories && totalCategory}
		<div class="sm:mx-4">
			<Table shadow>
				<TableHead>
					<TableHeadCell>Category</TableHeadCell>
					<TableHeadCell>Grade</TableHeadCell>
					<TableHeadCell>Weight</TableHeadCell>
					<TableHeadCell>Points</TableHeadCell>
				</TableHead>
				<TableBody>
					{#each gradeCategories.toSorted() as category}
						<TableBodyRow>
							<TableBodyCell>{category.name}</TableBodyCell>
							<TableBodyCell>
								{#if category.pointsEarned !== 0 || category.pointsPossible !== 0}
									{category.gradeLetter}
									{#if rawGradeCalcMatches}
										({Math.round(
											calculateGradePercentage(category.pointsEarned, category.pointsPossible) *
												1000
										) / 1000}%)
									{/if}
								{/if}
							</TableBodyCell>
							<TableBodyCell>{category.weightPercentage}%</TableBodyCell>
							<TableBodyCell>
								{category.pointsEarned} / {category.pointsPossible}
							</TableBodyCell>
						</TableBodyRow>
					{/each}
					<TableBodyRow>
						<TableBodyCell>Total</TableBodyCell>
						<TableBodyCell>
							{totalCategory.gradeLetter}
							{#if rawGradeCalcMatches}
								({Math.round(
									calculateGradePercentage(
										totalCategory.pointsEarned,
										totalCategory.pointsPossible
									) * 1000
								) / 1000}%)
							{/if}
						</TableBodyCell>
						<TableBodyCell />
						<TableBodyCell>
							{totalCategory.pointsEarned} / {totalCategory.pointsPossible}
						</TableBodyCell>
					</TableBodyRow>
				</TableBody>
			</Table>
		</div>
	{/if}

	{#if !rawGradeCalcMatches}
		<Alert class="m-4" color="red" border>
			<ExclamationCircleSolid slot="icon" size="sm" class="focus:outline-none" />

			<div class="flex flex-col gap-2">
				<button
					class="flex items-center"
					onclick={() => {
						calcWarningOpen = !calcWarningOpen;
					}}
				>
					<span class="font-bold text-left">
						{#if hypotheticalMode}
							Grade calculations in Hypothetical Mode are inaccurate
						{:else}
							Grade calculation error
						{/if}
					</span>
					{#if calcWarningOpen}
						<ChevronUpOutline size="md" class="focus" />
					{:else}
						<ChevronDownOutline size="md" class="focus" />
					{/if}
				</button>

				{#if calcWarningOpen}
					<span>
						Your class's official grade percentage does not match GradeVue's calculated grade
						percentage. This could mean that there are hidden assignments that GradeVue can't see,
						or that GradeVue isn't calculating your grade correctly. Your overall grade is still
						correct, but other things might be off.
					</span>
				{/if}
			</div>
		</Alert>
	{/if}

	<div class="flex flex-wrap justify-between items-center">
		<Checkbox bind:checked={hypotheticalMode} class="m-4">
			<div id="hypothetical-toggle" class="flex items-center mr-2">
				Hypothetical Mode
				<InfoCircleOutline size="sm" class="ml-2 focus:outline-none" />
			</div>
		</Checkbox>
		<Popover triggeredBy="#hypothetical-toggle" class="max-w-md">
			Hypothetical mode allows you to see what your grade would be if you got a certain score on an
			assignment.
		</Popover>

		{#if hypotheticalMode}
			<div transition:fade={{ duration: 200 }}>
				<Button color="light" class="mx-4" onclick={addHypotheticalAssignment}>
					<GridPlusOutline size="sm" class="mr-2 focus:outline-none" />
					Add Hypothetical Assignment
				</Button>
			</div>
		{/if}
	</div>

	{#if synergyAssignments.length > 0 || hypotheticalMode}
		<div transition:fade={{ duration: 200 }}>
			<Tabs class="m-4 mb-0" contentClass="p-4">
				<TabItem open title="All">
					<ol class="space-y-4">
						{#if hypotheticalMode}
							{#each reactiveAssignments as { gradePercentageChange, hidden, newHypothetical, date }, i}
								<li>
									<AssignmentCard
										bind:name={reactiveAssignments[i].name}
										bind:pointsEarned={reactiveAssignments[i].pointsEarned}
										bind:pointsPossible={reactiveAssignments[i].pointsPossible}
										bind:extraCredit={reactiveAssignments[i].extraCredit}
										gradePercentageChange={rawGradeCalcMatches ? gradePercentageChange : undefined}
										bind:notForGrade={reactiveAssignments[i].notForGrade}
										{hidden}
										showHypotheticalLabel={newHypothetical}
										bind:category={reactiveAssignments[i].category}
										categoryDropdownOptions={gradeCategories?.map((category) => category.name)}
										{date}
										editable={true}
										{recalculateGradePercentage}
									/>
								</li>
							{/each}
						{:else}
							{#each assignments as { name, pointsEarned, pointsPossible, extraCredit, gradePercentageChange, notForGrade, hidden, category, date }}
								<li>
									<AssignmentCard
										{name}
										{pointsEarned}
										{pointsPossible}
										{extraCredit}
										gradePercentageChange={rawGradeCalcMatches ? gradePercentageChange : undefined}
										{notForGrade}
										{hidden}
										{category}
										{date}
									/>
								</li>
							{/each}
						{/if}
					</ol>
				</TabItem>

				{#each [...new Set(realAssignments
							.map((assignment) => assignment.category)
							.toSorted())] as categoryName}
					<TabItem title={categoryName}>
						<ol class="space-y-4">
							{#if hypotheticalMode}
								{#each reactiveAssignments as { gradePercentageChange, hidden, newHypothetical, date, category: assignmentCategoryName }, i}
									{#if assignmentCategoryName === categoryName}
										<li>
											<AssignmentCard
												bind:name={reactiveAssignments[i].name}
												bind:pointsEarned={reactiveAssignments[i].pointsEarned}
												bind:pointsPossible={reactiveAssignments[i].pointsPossible}
												bind:extraCredit={reactiveAssignments[i].extraCredit}
												gradePercentageChange={rawGradeCalcMatches
													? gradePercentageChange
													: undefined}
												bind:notForGrade={reactiveAssignments[i].notForGrade}
												{hidden}
												showHypotheticalLabel={newHypothetical}
												bind:category={reactiveAssignments[i].category}
												categoryDropdownOptions={gradeCategories?.map((category) => category.name)}
												{date}
												editable={true}
												{recalculateGradePercentage}
											/>
										</li>
									{/if}
								{/each}
							{:else}
								{#each assignments as { name, pointsEarned, pointsPossible, extraCredit, gradePercentageChange, notForGrade, hidden, category, date }}
									{#if category === categoryName}
										<li>
											<AssignmentCard
												{name}
												{pointsEarned}
												{pointsPossible}
												{extraCredit}
												gradePercentageChange={rawGradeCalcMatches
													? gradePercentageChange
													: undefined}
												{notForGrade}
												{hidden}
												{date}
											/>
										</li>
									{/if}
								{/each}
							{/if}
						</ol>
					</TabItem>
				{/each}
			</Tabs>
		</div>
	{/if}

	<div class="top-12 left-0 w-full fixed md:top-0">
		<div class="absolute w-full md:pl-64">
			<div class="p-4 rounded-b-lg bg-gray-900 rounded flex justify-between">
				<span class="line-clamp-1 text-2xl">
					{courseName}
				</span>
				<span class="shrink-0 text-2xl flex items-center">
					{#if hypotheticalMode}
						{#if !categories && !rawGradeCalcMatches}
							<ExclamationCircleSolid class="mr-2 focus:outline-none" />
						{/if}
						{Math.round(hypotheticalGrade * 1000) / 1000}%
					{:else}
						{synergyCourse.Marks.Mark._CalculatedScoreString}
						{synergyCourse.Marks.Mark._CalculatedScoreRaw}%
					{/if}
				</span>
			</div>
		</div>
	</div>
{/if}
