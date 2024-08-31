<script lang="ts" module>
	import {
		type Query,
		type InstantClient,
		type Exactly,
		type LifecycleSubscriptionState,
		coerceQuery
	} from "@instantdb/core"

	const defaultState = {
		isLoading: true,
		data: undefined,
		pageInfo: undefined,
		error: undefined
	}

	export function useQuery<Q extends Query, Schema>(
		_core: InstantClient<Schema>,
		_query: Exactly<Query, Q> | null
	): { state: LifecycleSubscriptionState<Q, Schema>; query: any } {
		const query = $derived(_query ? coerceQuery(_query) : null)
		// @ts-expect-error
		let _state: LifecycleSubscriptionState<Q, Schema> = $state(defaultState)

		$effect(() => {
			if (!query) return

			const unsubscribe = _core.subscribeQuery<Q>(query, (result) =>
				Object.assign(_state, {
					...defaultState,
					...result,
					isLoading: !Boolean(result)
				})
			)

			return unsubscribe
		})

		return {
			state: _state,
			get query() {
				return query
			}
		}
	}
</script>

<script lang="ts">
	import { init, tx, id } from "@instantdb/core"

	const APP_ID = "32035a9c-a7ed-42d3-a71a-83eb8eca2570"

	interface Todo {
		id: string
		text: string
		done: boolean
		createdAt: number
	}

	type Schema = {
		todos: Todo
	}

	const db = init<Schema>({ appId: APP_ID })

	function addTodo(text: string) {
		db.transact(
			tx.todos[id()].update({
				text,
				done: false,
				createdAt: Date.now()
			})
		)
	}

	function deleteTodo(todo: Todo) {
		db.transact(tx.todos[todo.id].delete())
	}

	function toggleDone(todo: Todo) {
		db.transact(tx.todos[todo.id].update({ done: !todo.done }))
	}

	function deleteCompleted(todos: Todo[]) {
		const completed = todos.filter((todo) => todo.done)
		const txs = completed.map((todo) => tx.todos[todo.id].delete())
		db.transact(txs)
	}

	function toggleAll(todos: Todo[]) {
		const newVal = !todos.every((todo) => todo.done)
		db.transact(todos.map((todo) => tx.todos[todo.id].update({ done: newVal })))
	}
	const { state: subscription } = useQuery(db, { todos: {} })
</script>

{#snippet todoForm({ todos }: { todos: Todo[] })}
	<div class="form">
		<button class="toggleAll" onclick={() => toggleAll(todos)}>⌄</button>
		<input
			class="input"
			placeholder="What needs to be done?"
			type="text"
			onkeydown={(event) => {
				if (event.key !== "Enter") return
				addTodo(event.currentTarget.value)
				event.currentTarget.value = ""
			}}
		/>
	</div>
{/snippet}

{#snippet todoList({ todos }: { todos: Todo[] })}
	<div class="todoList">
		{#each todos as todo (todo.id)}
			<div class="todo" class:done={todo.done}>
				<input
					type="checkbox"
					class="checkbox"
					checked={todo.done}
					onchange={() => toggleDone(todo)}
				/>
				<div class="todoText">
					<span>{todo.text}</span>
				</div>
				<button onclick={() => deleteTodo(todo)} class="delete"> 𝘟 </button>
			</div>
		{/each}
	</div>
{/snippet}

{#snippet actionBar({ todos }: { todos: Todo[] })}
	<div class="actionBar">
		<div>Remaining todos: {todos.filter((todo) => !todo.done).length}</div>
		<button style="cursor: pointer" onclick={() => deleteCompleted(todos)}>Delete Completed</button>
	</div>
{/snippet}

{#if subscription.isLoading}
	<div>Fetching data...</div>
{:else if subscription.error}
	<div>Error fetching data: {subscription.error.message}</div>
{:else}
	{@const { todos } = subscription.data}
	<div class="container">
		<div class="header">todos</div>
		{@render todoForm({ todos })}
		{@render todoList({ todos })}
		{@render actionBar({ todos })}
		<div class="footer">Open another tab to see todos update in realtime!</div>
	</div>
{/if}

<style>
	button {
		all: unset;
	}

	.container {
		box-sizing: border-box;
		background-color: #fafafa;
		font-family: code, monospace;
		height: 100vh;
		display: flex;
		justify-content: center;
		align-items: center;
		flex-direction: column;
	}

	.header {
		letter-spacing: 2px;
		font-size: 50px;
		color: lightgray;
		margin-bottom: 10px;
	}

	.form {
		box-sizing: inherit;
		display: flex;
		border: 1px solid lightgray;
		border-bottom-width: 0px;
		width: 350px;
	}

	.toggleAll {
		font-size: 30px;
		cursor: pointer;
		margin-left: 11px;
		margin-top: -6px;
		width: 15px;
		margin-right: 12px;
	}

	.input {
		background-color: transparent;
		font-family: code, monospace;
		width: 287px;
		padding: 10px;
		font-style: italic;
	}

	.todoList {
		box-sizing: inherit;
		width: 350px;
	}

	.checkbox {
		font-size: 30px;
		margin-left: 5px;
		margin-right: 20px;
		cursor: pointer;
	}

	.todo {
		display: flex;
		align-items: center;
		padding: 10px;
		border: 1px solid lightgray;
		border-bottom-width: 0px;

		&.done .todoText span {
			text-decoration: line-through;
		}
	}

	.todoText {
		flex-grow: 1;
		overflow: hidden;
	}

	.delete {
		width: 25px;
		cursor: pointer;
		color: lightgray;
	}

	.actionBar {
		display: flex;
		justify-content: space-between;
		width: 328px;
		padding: 10px;
		border: 1px solid lightgray;
		font-size: 10px;
	}

	.footer {
		margin-top: 20px;
		font-size: 10px;
	}
</style>
