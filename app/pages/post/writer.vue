<template>
  <div class="post-create">
    <div class="draft-wrapper thin-scroll">
      <p class="title">
        创作夹
        <i class="el-icon-tickets"></i>
      </p>
      <div class="draft-list list-group">
        <loading v-if="draftLoading"></loading>
        <div v-if="drafts.length===0" class="none-wrapper text-center">
          <p>暂无数据，您可以创建草稿</p>
          <p>
            <i class="el-icon-caret-bottom"></i>
          </p>
        </div>
        <div
          @click="chooseDraft(dt)"
          class="list-group-item"
          :class="{'active':dt._id===post._id}"
          v-for="dt in drafts"
          :key="dt._id"
        >
          <span class="d-title">{{dt.title}}</span>
          <p class="create-date">
            创建时间：{{ dt.createAt | formatDate }}
            <el-badge
              :class="{
            'color-success':dt.published,
            'color-warn':!dt.published,
            }"
            >{{dt.published===true?'已发布':'未发布'}}</el-badge>
          </p>
        </div>
      </div>
      <div class="text-center mt-2 p-2">
        <p @click="createDrafts" class="create-btn el-button btn-block">
          <i class="el-icon-plus"></i> 创建草稿
        </p>
      </div>
    </div>
    <div class="editor-wrapper">
      <div class="row-input d-flex justify-content-between">
        <div class="input-wrapper">
          <input v-model="post.title" placeholder="请输入文章标题" type="text" class="input-title">
        </div>
        <div class="menu-wrapper">
          <span
            class="cursor-pointer"
            :class="{'active':editor.show}"
            @click="editor.show=!editor.show"
          >
            <i class="el-icon-rank"></i> 预览
          </span>
        </div>
      </div>
      <textarea
        placeholder="请输入文档摘要"
        v-model="post.desc"
        name
        class="input-content input-zy content-wrapper thin-scroll"
      ></textarea>
      <textarea
        placeholder="请输入文章内容,支持截图粘贴图片<code>CTRL+V</code>"
        v-paste-img="pasteImg"
        v-model="post.content"
        name
        class="input-content content-wrapper thin-scroll"
      ></textarea>
      <div class="tags-wrapper">
        <div class="tag-list">
          <div class="el-tag mr-2" v-for="(t,i) in post.tags" :key="i">
            <span>
              {{t}}
              <i @click="removeTag(t)" class="el-icon-close"></i>
            </span>
          </div>
        </div>
        <div class="tag-input-wrapper">
          <input v-model="newTag.tag" placeholder="创建标签" type="text" class="tag-input">
          <button @click="createTag" class="submit-tag">确定</button>
        </div>
      </div>
      <div class="footer-wrapper">
        <div class="left-f">
          <el-select v-model="post._categoryId" placeholder="请选择发布位置">
            <el-option value="个人专栏">个人专栏</el-option>
          </el-select>
        </div>
        <div class="right-f">
          <el-button @click="save" type="info">保存</el-button>
          <el-button :disabled="publishing" @click="publish" type="primary">
            <i class="el-icon-loading" v-if="publishing"></i>发布文章
          </el-button>
        </div>
      </div>
    </div>
    <div v-if="editor.show" class="review-wrapper thin-scroll">
      <h3 class="title">{{post.title}}</h3>
      <div>
        <span class="desc">{{post.desc}}</span>
      </div>
      <vue-markdown
        class="markdown-body"
        :source="post.content"
        :show="editor.show"
        :html="editor.html"
        :breaks="editor.breaks"
        :linkify="editor.linkify"
        :emoji="editor.emoji"
        :typographer="editor.typographer"
        :toc="editor.toc"
        toc-id="editor.toc"
      ></vue-markdown>
    </div>
  </div>
</template>

<script>
import DayJs from "dayjs";
import Loading from "../../components/Loading";
import VueMarkdown from "vue-markdown";
import _ from "lodash";

/**
 * 咱贴上传图片
 */
// $("#post_content").on('paste', function (ev) {
//
// });
export default {
  name: "create",
  components: {
    Loading,
    VueMarkdown
  },
  head() {
    return {
      title: "写文章",
      links: [],
      scripts: []
    };
  },
  middleware: ["base", "auth"],
  data() {
    return {
      draftLoading: true,
      publishing: false,
      drafts: [], //草稿
      post: {
        _id: "",
        title: "",
        desc: "", //摘要
        content: "",
        pics: [],
        tags: [],
        version: "0.0.1"
      },
      newTag: {
        creating: true,
        tag: ""
      },
      editor: {
        show: true,
        html: false,
        breaks: true,
        linkify: true,
        emoji: true,
        typographer: true,
        toc: false
      }
    };
  },
  watch: {
    "post.content"(n) {
      //this.post.content=xxs
    }
  },
  methods: {
    //粘贴成功后绑定一个事件
    async pasteImg(res) {
      if (res.status === 200 && res.data) {
        const data = res.data;
        if (!data.ok) return false;
        //然后注入到内容里面
        this.post.content += `\n![${data.data.file.filename}](${
          data.data.url
        })`;
      }
    },
    async save() {},
    /*发布草稿*/
    async publish() {
      //字段验证
      if (this.post.tags.length === 0) {
        this.$message.error("请添加至少1个标签");
        return false;
      }
      if (this.post.content.length <= 50) {
        this.$message.error("请再写点内容吧");
        return false;
      }
      this.publishing = true;
      //解析内容中的图片数组
      let pics = this.post.content.match(/\!\[.+\]\(.+\)/g);
      //交给后台解析吧
      let res = await this.$axios.put(
        `post/${this.post._id}/publish`,
        this.post
      );
      if (res.code !== Codes.FAIL) {
        this.$notify.success("发布成功");
      }
      this.publishing = false;
    },
    async remove() {},
    //创建标签
    async createTag() {
      //判断长度
      let tagLen = this.newTag.tag.length;
      if (tagLen <= 0 || tagLen > 30) {
        this.$message.error("标签长度0~30位字符");
        return false;
      }
      //判断postTags
      let pTagCount = this.post.tags.length;
      if (pTagCount > 5) {
        this.$message.error("最多添加3个标签");
        return false;
      }
      //判断重复
      if (_.findIndex(this.post.tags, x => x === this.newTag.tag) !== -1) {
        this.$message.error("已经存在的标签");
        return false;
      }
      this.newTag.creating = true;
      let res = await this.$axios.post(`tag/create`, this.newTag);
      this.post.tags.push(this.newTag.tag);
      this.newTag.tag = "";
      this.newTag.creating = false;
    },
    //从Post里面移除一个Tag
    removeTag(tag) {
      let ix = _.findIndex(this.post.tags, x => x === tag);
      this.post.tags.splice(ix, 1);
    },
    /**
     * 选择草稿
     * @param draft
     * @returns {Promise<void>}
     */
    async chooseDraft(draft) {
      if (false === _.isArray(draft.tags)) {
        draft.tags = ["默认标签"];
      }
      this.post = { ...draft };
    },
    /**
     * 创建一个草稿
     * @returns {Promise<void>}
     */
    async createDrafts() {
      let draft = {
        title: DayJs().format("YYYY-MM-DD"),
        createAt: DayJs().format("YYYY-MM-DD HH:mm")
      };
      //创建一个草稿
      draft = await this.$axios.post(`posts/create`, {
        published: false,
        ...draft
      });
      this.loadDrafts();
    },
    /**
     * 获取草稿
     * @returns {Promise<void>}
     */
    async loadDrafts() {
      this.draftLoading = true;
      //获取草稿箱
      let res = await this.$axios.get(`posts/drafts`);
      this.draftLoading = false;
      if (!res.ok) {
        return;
      }
      const drafts = res.data;
      this.drafts = [...drafts];
      if (drafts.length > 0) {
        this.chooseDraft(drafts[0]);
      } else {
        //如果没有的话创建一个
        await this.createDrafts();
        this.loadDrafts();
      }
    }
  },
  created() {
    this.loadDrafts();
  }
};
</script>

<style scoped lang="scss">
.post-create {
  display: flex;
  flex-direction: row;
  position: fixed;
  width: 100%;
  padding: 1rem 0.5rem;
  bottom: 0;
  top: 65px;
}

.draft-wrapper {
  width: 300px;
  background: #fff;
  overflow-y: auto;
  .title {
    font-size: 20px;
    line-height: 30px;
    padding: 1rem 2rem;
    text-align: left;
  }
  .create-btn {
    text-align: center;
    cursor: pointer;
    font-size: 16px;
    &:hover {
      color: $blue-500;
    }
  }
  .draft-list {
    .list-group-item {
      border-right: 0;
      border-left: 0;
      border-radius: 0;
      cursor: pointer;
      line-height: 50px;
      border-left: 8px solid transparent;
      transition: 0.25s all;
      &:hover,
      &.active {
        background: transparent;
        color: #000;
        border-bottom: 1px solid rgba(0, 0, 0, 0.125);
        border-top: 1px solid rgba(0, 0, 0, 0.125);
        border-radius: 0;
      }
      &:hover {
        background: $grey-50;
      }
      &.active {
        border-left: 8px solid $blue-500;
      }
      .d-title {
        font-size: 18px;
      }
      .create-date {
        line-height: 15px;
        font-size: 14px;
        color: $grey-600;
      }
    }
  }
  .none-wrapper {
    color: $grey-500;
  }
}

.editor-wrapper {
  margin-left: 1rem;
  background: #fff;
  flex: 1;
  padding: 1rem;
  display: flex;
  flex-direction: column;
  transition: 0.3s all;
  .content-wrapper {
    margin: 1rem 0;
    flex: 1 1 auto;
    flex-direction: column;
    overflow-y: auto;
  }
  .row-input {
    border-bottom: 1px solid $grey-200;
    padding: 3px 1rem;
    line-height: 50px;
    flex: none;
    display: block;
    .input-wrapper {
      display: block;
    }
    .input-title {
      width: 100%;

      border: none;
      font-size: 20px;
    }
    .menu-wrapper {
      span.active {
        color: $blue-500;
      }
    }
  }

  .input-content {
    width: 100%;
    height: 100%;
    display: block;
    flex: auto;
    padding: 0 1rem;
    border: none;
    border-bottom: 1px solid $grey-200;
    resize: none;
    &.input-zy {
      height: 120px;
    }
  }
  .footer-wrapper {
    border-top: 1px solid $grey-200;
    padding-top: 1rem;
    display: flex;
    flex: none;
    flex-direction: row;
    justify-content: space-between;
  }
}

.tags-wrapper {
  height: 50px;
  flex: none;
  display: flex;
  width: 100%;
  flex-direction: row;
  justify-content: space-between;
  .tag-list {
    display: flex;
    width: 100%;
    flex-direction: row;
  }
  .tag-input {
    border: none;
    border-bottom: 1px solid $grey-200;
    padding: 5px;
    width: 160px;
    flex: none;
    text-align: left;
  }
  .tag-input-wrapper {
    position: relative;
    .submit-tag {
      position: absolute;
      right: 0;
      border: 1px solid $grey-200;
      background: $blue-500;
      color: #fff;
      cursor: pointer;
      line-height: 28px;
    }
  }
}

.review-wrapper {
  flex: 1;
  margin: 0 0 0 1rem;
  background: #fff;
  padding: 1rem;
  overflow-y: auto;
  &.active {
  }
  .title {
    line-height: 50px;
    border-bottom: 1px solid $grey-200;
  }
  .desc {
    font-size: 13px;
    color: $grey-500;
    margin: 1rem;
    white-space: pre-wrap;
    width: 100%;
    overflow: hidden;
    line-height: 1.6;
    word-wrap: break-word;
  }
  .markdown-body {
    margin-top: 1rem;
    font-size: 16px;
  }
}
</style>
