<template>
  <div class="page-container">
    <div class="content-container">
      <div class="input-wrapper">
        <input
          v-model="searchStr"
          class="search-input"
          type="search"
          @input="search"
        />
      </div>

      <div class="list-wrapper">
        <ul v-if="usersSlice.length">
          <user-item
            v-for="user in usersSlice"
            :key="`user#${user.email}`"
            :user="user"
            :search-str="searchStr"
            :suitable="suitableUsers.includes(user.email)"
            :toggle-selection="toggleSelection"
          />

          <infinite-loading
            ref="infiniteLoadingRef"
            spinner="spiral"
            @infinite="loadMore"
          >
            <span slot="no-more" class="infinite-end">No more results.</span>
          </infinite-loading>
        </ul>

        <ul v-else-if="users.length">
          <span class="infinite-end">No results.</span>
        </ul>
      </div>
    </div>
  </div>
</template>

<script>
import UserItem from '@/components/UserItem'

export default {
  name: 'UserListPage',
  components: { UserItem },
  props: {
    searchParam: {
      type: String,
      required: false,
      default: '',
    },
  },
  data() {
    return {
      users: [],
      usersSlice: [],
      suitableUsers: [],
      searchStr: this.searchParam,
      searchStopIndex: 0,
      page: 1,
      pageSize: 50,
    }
  },
  async created() {
    this.users = await fetch('/data/users.json').then((res) => res.json())
    if (this.searchParam) {
      this.search()
    } else {
      this.usersSlice = this.users.slice(0, this.pageSize)
    }
  },
  methods: {
    /**
     * Filter items that contain the searchStr from usersArr until result array length reaches pageSize
     */
    filterResults(usersArr) {
      const newSlice = []
      let stopIndex = 0

      usersArr.some((user, index) => {
        const containsSearchStr = Object.keys(user).some((key) => {
          if (key === 'avatar') return false
          return user[key].toLowerCase().includes(this.searchStr.toLowerCase())
        })

        if (containsSearchStr) {
          newSlice.push(user)
        }

        stopIndex = index + 1
        return newSlice.length === this.pageSize
      })

      return [newSlice, stopIndex]
    },
    /**
     * Search listener
     */
    search() {
      const [newSlice, stopIndex] = this.filterResults(this.users)

      document.querySelector('.list-wrapper').scrollTo(0, 0)

      this.page = 1
      this.usersSlice = newSlice
      this.searchStopIndex = stopIndex

      if (this.$refs.infiniteLoadingRef) {
        this.$refs.infiniteLoadingRef.stateChanger.reset()
      }
    },
    /**
     * Infinite scroll listener
     */
    loadMore($state) {
      setTimeout(() => {
        this.page++

        if (!this.searchStr) {
          const newPage = this.users.slice(
            this.pageSize * (this.page - 1),
            this.pageSize * this.page
          )

          if (newPage.length > 0) {
            this.usersSlice = this.usersSlice.concat(newPage)
          } else {
            $state.complete()
          }
        } else {
          // search from searchStopIndex
          const [newSlice, stopIndex] = this.filterResults(
            this.users.slice(this.searchStopIndex)
          )

          if (newSlice.length > 0) {
            this.searchStopIndex += stopIndex
            this.usersSlice = this.usersSlice.concat(newSlice)
          } else {
            $state.complete()
          }
        }

        $state.loaded()
      }, 500)
    },
    /**
     * Toggle user selection
     */
    toggleSelection(userEmail) {
      if (this.suitableUsers.includes(userEmail)) {
        const index = this.suitableUsers.indexOf(userEmail)
        this.suitableUsers.splice(index, 1)
      } else {
        this.suitableUsers.push(userEmail)
      }
    },
  },
}
</script>

<style lang="scss" scoped>
.page-container {
  background-color: #eee;
  height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 20px;

  .content-container {
    background-color: #fff;
    max-width: 564px;
    width: 100%;
    margin: 34px 0 45px;
    padding: 25px 12px 0 10px;
    overflow: hidden;
    position: relative;
    flex-grow: 1;

    .input-wrapper {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      padding: 1.1875em 1.75em 0 0.75em;
      z-index: 2;
      font-size: 1rem;

      &:before {
        content: '';
        height: 1.5em;
        width: 1.5em;
        mask-image: url(/images/search.svg);
        background: #000;
        opacity: 0.54;
        position: absolute;
        top: 1.9375em;
        left: 1.75em;
      }

      .search-input {
        width: 100%;
        background: #fafafa;
        border: none;
        border-radius: 2px;
        box-shadow: 0 0 2px rgba(0, 0, 0, 0.12), 0 2px 2px rgba(0, 0, 0, 0.24);
        font-size: 1.5em;
        line-height: 1.1667em;
        color: rgba(0, 0, 0, 0.75);
        padding: 0.5em 0.4167em 0.2917em 2.0833em;
        outline: none;

        &::-webkit-search-cancel-button {
          -webkit-appearance: none;
        }
      }
    }

    .list-wrapper {
      height: 100%;
      overflow: auto;
      padding-right: 10px;

      ul {
        margin: 0;
        padding: 0 2px;
        padding-top: 3.875rem;
        list-style: none;
      }

      .infinite-end {
        font-size: 12px;
        color: rgba(0, 0, 0, 0.5);
        margin-bottom: 20px;
        display: block;
        text-align: center;
      }
    }
  }

  @media screen and (max-width: 768px) {
    padding: 15px;

    .content-container {
      margin: 0;

      .input-wrapper {
        font-size: 0.85rem;
      }

      .list-wrapper ul {
        padding-top: 3.29375rem;
      }
    }
  }
}

::-webkit-scrollbar {
  width: 4px;
}

::-webkit-scrollbar-track {
  width: 1px;
  border: 1px solid white;
  border-left-width: 2px;
  background-color: rgba(0, 0, 0, 0.16);
  margin-bottom: 10px;
}

::-webkit-scrollbar-thumb {
  background: #4d4d4d;
  border-radius: 2px;
}
</style>
