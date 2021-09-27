<template>
  <div>
    <h1>develop環境</h1>
    <amplify-authenticator>
      <div style="max-width: 800px; margin-top: 20px">
        <v-text-field
          v-model="form.comment"
          label="コメント"
          placeholder="ここにコメントを書きましょう"
          outlined
          class="mx-auto"
          append-icon="mdi-check-bold"
          style="max-width: 100%; box-sizing: border-box"
          @keydown="onEnter"
          @click:append="createChat"
        ></v-text-field>

        <v-card v-for="(item, index) in items" :key="index" tile>
          <v-list-item two-line>
            <v-list-item-content>
              <v-list-item-title>{{ item.comment }}</v-list-item-title>
              <v-list-item-subtitle>by: {{ item.owner }}</v-list-item-subtitle>
            </v-list-item-content>
          </v-list-item>
        </v-card>
      </div>

      <v-card>
        <amplify-sign-out v-if="logoutBtn"></amplify-sign-out>
      </v-card>
    </amplify-authenticator>
  </div>
</template>
<script lang="ts">
import { Component, Vue } from 'nuxt-property-decorator'
import { API, graphqlOperation } from 'aws-amplify' // Amplifyライブラリを読み込み
import { onAuthUIStateChange } from '@aws-amplify/ui-components'
import { createChat } from '~/src/graphql/mutations' // GraphQL Mutation（データをエンドポイントに送信する構文?）
import { listChats } from '~/src/graphql/queries' // GraphQL Query（データを読み込む構文？）
import { onCreateChat } from '~/src/graphql/subscriptions'
interface Form {
  comment: string
}
// アノテーション
@Component
export default class Index extends Vue {
  private items: { comment: string; owner: string }[] = []

  private form: Form = {
    comment: '',
  }

  private username: string = ''

  private logoutBtn: boolean = false

  private created(): void {
    this.getChatList()
    // this.subscribe()
    onAuthUIStateChange((authState: any, authData: any): void => {
      // ログインステータスが変化したとき
      if (authState === 'signedin') {
        // ログインした場合
        this.username = authData.username
        this.getChatList() // 一覧呼び出し
        this.subscribe() // GraphQL Subscription
        this.logoutBtn = true
      } else {
        this.items = [] // ログアウトしたときなどは一覧を削除
        this.logoutBtn = false
      }
    })
  }

  private async createChat(): Promise<void> {
    const comment: string = this.form.comment
    if (!comment) return
    const chat = { comment }
    await API.graphql({
      query: createChat,
      variables: { input: chat },
    })
    this.form.comment = ''
  }

  private onEnter(event: any) {
    if (event.keyCode !== 13) return
    this.createChat()
  }

  private async getChatList(): Promise<void> {
    const chatList: any = await API.graphql({
      query: listChats,
    })
    this.items = chatList.data.listChats.items
  }

  private subscribe(): any {
    const observable = API.graphql(
      graphqlOperation(onCreateChat, { owner: this.username })
    ) as any
    observable.subscribe({
      next: (eventData: any) => {
        // コメントが送信されて追加されたとき、送信内容を一覧に追加
        const chat: any = eventData.value.data.onCreateChat // データを読み込み
        if (this.items.some((item) => item.comment === chat.comment)) return // すでに表示されているデータは無視
        this.items = [...this.items, chat] // 新しいデータを追加
      },
    })
  }
}
</script>
