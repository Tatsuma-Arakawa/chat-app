<template>
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
</template>
<script lang="ts">
import { Component, Vue } from 'nuxt-property-decorator'
import { API } from 'aws-amplify' // Amplifyライブラリを読み込み
import { createChat } from '~/src/graphql/mutations' // GraphQL Mutation（データをエンドポイントに送信する構文?）
import { listChats } from '~/src/graphql/queries' // GraphQL Query（データを読み込む構文？）
import { onCreateChat } from '~/src/graphql/subscriptions'
interface Form {
  comment: string
}
// アノテーション
@Component
export default class Index extends Vue {
  public created() {
    this.getChatList()
    this.subscribe()
  }

  form: Form = {
    comment: '',
  }

  items: { comment: string; owner: string }[] = []

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

  private async getChatList(): Promise<void> {
    // コメント一覧を取得
    const chatList: any = await API.graphql({
      query: listChats, // GraphQL Query
    })
    this.items = chatList.data.listChats.items // 読み込みしたデータを一覧に表示
  }

  private subscribe(): any {
    const observable = API.graphql({ query: onCreateChat }) as any
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
