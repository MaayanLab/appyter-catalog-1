<script>
  import { Modal } from 'bootstrap'
  import auth from '@/stores/keycloak_auth_store'
  import Loader from '@/fragments/Loader.svelte'
  import url_for from '@/utils/url_for'

  const base_url = window.location.origin

  let modalRef
  let bsModal
  let prompt_delete

  $: if (modalRef) {
    bsModal = new Modal(modalRef)
    modalRef.addEventListener('hidden.bs.modal', () => prompt_delete = undefined)
  }

  $: if (bsModal !== undefined) {
    if (prompt_delete !== undefined) {
      bsModal.show()
    } else {
      bsModal.hide()
    }
  }

  let notebooks
  let offset = 0
  let limit = 10
  let count

  async function load_notebooks({ offset: _offset, limit: _limit }) {
    notebooks = undefined
    const res = await fetch(url_for({
      path: `${base_url}/postgrest/user_instance`,
      params: {
        offset: _offset,
        limit: _limit,
      },
    }), {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
        'Prefer': 'count=estimated',
        'Authorization': `Bearer ${await $auth.keycloak.getValidToken()}`,
      },
    })
    offset = _offset
    limit = _limit
    notebooks = await res.json()
    if (notebooks.length < limit) {
      count = offset + notebooks.length
    } else {
      count = Number(res.headers.get('Content-Range').split('/')[1])
    }
  }

  async function delete_notebook(id) {
    const res = await fetch(url_for({
      path: `${base_url}/postgrest/user_instance`,
      params: {
        id: `eq.${id}`,
      },
    }), {
      method: 'DELETE',
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
        'Authorization': `Bearer ${await $auth.keycloak.getValidToken()}`,
      },
    })
    if (res.status === 204) {
      if (offset >= count-1) {
        // deletion would cause this page to disappear, move back a page
        offset = Math.max(0, offset - limit)
      }
      await load_notebooks({ offset, limit })
    } else {
      console.error(await res.text())
    }
  }

  if ($auth.state === 'auth') {
    load_notebooks({ offset, limit }).catch(e => console.error(e))
  }
</script>

<div class="container flex-grow-1">
  <h1>Notebooks</h1>

  <div class="d-flex justify-content-end py-2">
    <a
      class="btn bg-success text-white mx-1"
      href="/#/"
    >Create</a>
    <button
      class="btn bg-primary text-white mx-1"
      on:click={() => load_notebooks({ offset, limit }).catch(e => console.error(e))}
    >Refresh</button>
  </div>

  <table class="table table-striped table-fixed">
    <tr>
      <th scope="col">Instance</th>
      <th scope="col">Appyter</th>
      <th scope="col">Version</th>
      <th scope="col">Created</th>
      <th scope="col">Actions</th>
    </tr>
    {#if notebooks === undefined}
      <tr>
        <td class="text-center" colspan="100%"><Loader /></td>
      </tr>
    {:else if notebooks.length === 0}
      <tr>
        <td class="text-center" colspan="100%">No notebooks found</td>
      </tr>
    {:else}
      {#each notebooks as notebook}
        <tr>
          <td><a href={url_for({
            path: `/${notebook.instance.metadata.appyter.info.name}/${notebook.instance.id}/`,
            params: {
              storage: notebook.instance.metadata.storage,
              executor: notebook.instance.metadata.executor,
            },
          })}>{notebook.instance.id.slice(0, 8)}...</a></td>
          <td><a href="/#/{notebook.instance.metadata.appyter.info.name}">{notebook.instance.metadata.appyter.info.title}</a></td>
          <td>{notebook.instance.metadata.appyter.info.version}</td>
          <td>{notebook.ts}</td>
          <td class="text-center"><button
            class="btn btn-sm bg-danger text-white"
            on:click={() => {
              prompt_delete = notebook
            }}
          >Delete</button></td>
        </tr>
      {/each}
      <tr>
        <td colspan="100%">
          {#if !count}
            <div class="text-center">
              No notebooks found.
            </div>
          {:else}
            <ul class="pagination">
              <li class="page-item">
                <button
                  class="btn page-link" style="background-color: inherit;"
                  aria-label="Previous"
                  class:disabled={offset === 0}
                  on:click={evt => {
                    load_notebooks({ offset: Math.max(0, offset - limit), limit }).catch(e => console.error(e))
                  }}
                >
                  <span aria-hidden="true">&laquo;</span>
                </button>
              </li>
              <li class="page-item">
                <span class="page-link text-black" style="background-color: inherit; border: 0;" aria-label="Next">
                  Showing notebooks {offset+1} - {offset + notebooks.length} of {count}
                </span>
              </li>
              <li class="page-item">
                <button
                  class="btn page-link" style="background-color: inherit;"
                  class:disabled={offset + limit >= count}
                  on:click={evt => {
                    load_notebooks({ offset: Math.min(count - 1, offset + limit), limit }).catch(e => console.error(e))
                  }}
                  aria-label="Next"
                >
                  <span aria-hidden="true">&raquo;</span>
                </button>
              </li>
            </ul>
          {/if}
        </td>
      </tr>
    {/if}
  </table>

  <div class="alert alert-primary">
    These are notebooks you've saved. Deleting a notebook from this list will unlink it from your account it will then be subject for deletion as long as no other users have it linked.
  </div>
</div>


<div bind:this={modalRef} class="modal fade">
  <div class="modal-dialog">
    <div class="modal-content">
      <div class="modal-header">
        <h5 class="modal-title">Are you sure you want to delete this notebook?</h5>
      </div>
      <div class="modal-body">
        {#if prompt_delete !== undefined}
        <b>Instance:</b>&nbsp;<a href={url_for({
          path: `/${prompt_delete.instance.metadata.appyter.info.name}/${prompt_delete.instance.id}/`,
          params: {
            storage: prompt_delete.instance.metadata.storage,
            executor: prompt_delete.instance.metadata.executor,
          },
        })}>{prompt_delete.instance.id}</a><br />
        <b>Appyter:</b>&nbsp;<a href="/#/{prompt_delete.instance.metadata.appyter.info.name}">{prompt_delete.instance.metadata.appyter.info.title}</a> v{prompt_delete.instance.metadata.appyter.info.version}<br />
        <b>Created:</b>&nbsp;{prompt_delete.ts}
        {/if}
      </div>
      <div class="modal-footer">
        <button type="button" class="btn btn-danger" on:click={() => {
          delete_notebook(prompt_delete.id).catch(e => console.error(e))
          prompt_delete = undefined
        }}>Delete</button>
        <button type="button" class="btn btn-secondary" on:click={() => {
          prompt_delete = undefined
        }}>Cancel</button>
      </div>
    </div>
  </div>
</div>
