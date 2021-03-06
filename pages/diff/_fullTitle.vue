<template lang="pug">
.page.page-diff
  .diff-content(v-html="diffHtml")
</template>

<script>
import { createPatch } from 'diff'
import articleManager from '~/utils/articleManager'
import { Diff2Html } from 'diff2html'

export default {
  validate ({ params, query, store }) {
    return Number(query.old) && Number(query.new)
  },
  async asyncData ({ params, query, req, res, error, store }) {
    store.commit('meta/clear')
    const fullTitle = params.fullTitle
    store.commit('meta/update', {
      title: `"${fullTitle}" 두 판 차이 보기`
    })
    try {
      const article = await articleManager.getByFullTitle(fullTitle, {
        fields: ['fullTitle', 'allowedActions'],
        req,
        res
      })
      store.commit('meta/update', {
        title: `"${article.fullTitle}" 두 판 차이 보기`,
        toolBox: {
          allowedActions: article.allowedActions,
          fullTitle: article.fullTitle
        }
      })
      const [oldRevision, newRevision] = await Promise.all([
        articleManager.getRevision({
          id: Number(query.old),
          req,
          res
        }),
        articleManager.getRevision({
          id: Number(query.new),
          req,
          res
        })
      ])
      const diffHtml = Diff2Html.getPrettySideBySideHtmlFromDiff(
        createPatch(
          article.fullTitle,
          oldRevision.wikitext,
          newRevision.wikitext,
          `${oldRevision.id}판`,
          `${newRevision.id}판`
        )
      )
        .replace('<span class="d2h-tag d2h-changed d2h-changed-tag">CHANGED</span>', '')
        .replace('<span class="d2h-tag d2h-moved d2h-moved-tag">RENAMED</span>', '')
      return {
        oldRevision,
        newRevision,
        diffHtml
      }
    } catch (err) {
      if (!err.response) {
        return error({ statusCode: 500 })
      }
      if (err.response.status === 404) {
        return error({ statusCode: 404, message: '문서가 존재하지 않습니다.' })
      }
      if (err.response.data.name === 'UnauthorizedError') {
        return error({ statusCode: 403, message: '권한이 없습니다.' })
      }
      return error({ statusCode: 500 })
    }
  }
}
</script>
