<template>
  <UModal
    v-model:open="isOpen"
    :title="state.id ? '编辑站点' : '新增站点'"
    :dismissible="false"
    :ui="{ footer: 'justify-end' }"
    aria-describedby="website-modal"
  >
    <template #body>
      <UForm ref="form" :schema="schema" :state="state" class="space-y-4" @submit="onSubmit">
        <UFormField label="所属分类" name="category_id" required>
          <USelect
            v-model="state.category_id"
            placeholder="请选择所属分类"
            value-key="id"
            label-key="name"
            :items="categoryList ?? []"
            class="w-full"
          />
        </UFormField>

        <UFormField label="站点名称" name="name" required>
          <UInput
            v-model="state.name"
            placeholder="请输入站点名称"
            class="w-full"
            size="lg"
            maxlength="12"
            aria-describedby="character-count"
          >
            <template #trailing>
              <div
                id="character-count"
                class="text-xs text-muted tabular-nums"
                aria-live="polite"
                role="status"
              >
                {{ state.name?.length || 0 }}/12
              </div>
            </template>
          </UInput>
        </UFormField>

        <UFormField label="网站链接" name="url" required>
          <UInput
            v-model="state.url"
            placeholder="请输入网站链接"
            class="w-full"
            size="lg"
            aria-describedby="url"
          />
        </UFormField>

        <UFormField label="Logo" name="logo" required>
          <UInput
            v-model="state.logo"
            placeholder="请输入Logo"
            class="w-full"
            size="lg"
            aria-describedby="logo"
          />
        </UFormField>

        <UFormField label="图标颜色" name="color">
          <UInput
            v-model="state.color"
            placeholder="请输入图标颜色"
            class="w-full"
            size="lg"
            aria-describedby="color"
          />
        </UFormField>

        <div class="grid grid-cols-4 gap-4">
          <UFormField label="置顶" name="pinned">
            <USwitch unchecked-icon="i-lucide-x" checked-icon="i-lucide-check" v-model="state.pinned" />
          </UFormField>
          <UFormField label="VPN" name="vpn">
            <USwitch unchecked-icon="i-lucide-x" checked-icon="i-lucide-check" v-model="state.vpn" />
          </UFormField>
          <UFormField label="推荐" name="recommend">
            <USwitch unchecked-icon="i-lucide-x" checked-icon="i-lucide-check" v-model="state.recommend" />
          </UFormField>
          <UFormField label="常用" name="commonlyUsed">
            <USwitch unchecked-icon="i-lucide-x" checked-icon="i-lucide-check" v-model="state.commonlyUsed" />
          </UFormField>
        </div>

        <UFormField name="desc" label="站点描述">
          <UTextarea
            v-model="state.desc"
            placeholder="请输入站点描述"
            :maxrows="4"
            autoresize
            class="w-full"
            maxlength="100"
            size="lg"
          />
        </UFormField>

        <UFormField name="tags" label="站点标签" required>
          <UInputTags
            v-model="state.tags"
            :max-length="4"
            placeholder="请输入站点标签"
            size="lg"
            icon="ri:price-tag-3-line"
            class="w-full"
          />
        </UFormField>

        <UFormField name="sort" label="排序" required>
          <UInputNumber v-model="state.sort" placeholder="请输入排序" class="w-full" size="lg" :min="1" :max="9999" />
        </UFormField>
      </UForm>
    </template>

    <template #footer>
      <UButton
        color="neutral"
        variant="outline"
        label="取消"
        class="cursor-pointer"
        @click="closeModal"
        size="lg"
        :disabled="loading"
      />
      <UButton
        color="neutral"
        variant="solid"
        label="确认"
        :loading="loading"
        class="cursor-pointer"
        @click="form?.submit()"
        size="lg"
        :disabled="loading"
      />
    </template>
  </UModal>
</template>

<script setup lang="ts">
import { z } from "zod";
import type { WebsiteEdit, CategoryList } from "~/lib/type";
import { RESPONSE_STATUS_CODE } from "~/lib/enum";
import type { FormSubmitEvent } from "@nuxt/ui";
import { reactive, ref, computed, watch } from "vue";

const props = defineProps<{
  modelValue: boolean;
  website?: WebsiteEdit;
  categoryList?: CategoryList[];
}>();

const emit = defineEmits<{
  "update:modelValue": (value: boolean) => void;
  success: () => void;
}>();

const form = useTemplateRef("form");
const toast = useToast();

const hexColorRegex = /^#([A-Fa-f0-9]{6}|[A-Fa-f0-9]{3})$/;

// 使用 z.preprocess 去除前后空格并允许空字符串转为 undefined，方便 color 字段可空
const schema = z.object({
  id: z.string().optional(),
  category_id: z.string().min(1, "请选择所属分类"),
  name: z
    .string()
    .min(1, "站点名称不能为空")
    .max(12, "站点名称不能超过12个字符")
    .trim(),
  url: z.string().url("请输入正确的站点链接").trim(),
  logo: z.string().min(1, "请输入Logo").trim(),
  color: z
    .string()
    .optional()
    .transform((val) => (val?.trim() === "" ? undefined : val))
    .refine((val) => val === undefined || hexColorRegex.test(val), {
      message: "必须是有效的 HEX 颜色（如 #FF0000 或 #FFF），或留空",
    }),
  tags: z
  .array(
    z.preprocess(
      (val) => (typeof val === "string" ? val.trim() : val),
      z.string().min(1, "标签不能为空")
    )
  )
  .min(1, "请输入站点标签"),
  pinned: z.boolean().optional(),
  vpn: z.boolean().optional(),
  recommend: z.boolean().optional(),
  commonlyUsed: z.boolean().optional(),
  desc: z
    .string()
    .max(100, "站点描述不能超过100个字符")
    .optional()
    .nullable(),
  icon: z.string().max(50, "站点图标不能超过50个字符").optional(),
  sort: z.number().min(1).max(9999),
});

type Schema = z.infer<typeof schema>;

// 初始化状态，避免 Partial 导致类型弱化
const getDefaultState = (): Schema => ({
  id: undefined,
  category_id: "",
  name: "",
  url: "",
  logo: "",
  color: undefined,
  tags: [],
  pinned: false,
  vpn: false,
  recommend: false,
  commonlyUsed: false,
  desc: undefined,
  icon: undefined,
  sort: 1,
});

const state = reactive<Schema>(getDefaultState());
const loading = ref(false);

const isOpen = computed({
  get: () => props.modelValue,
  set: (val) => emit("update:modelValue", val),
});

// 监听 props.website，避免直接赋值导致响应式丢失，改为逐字段赋值
watch(
  () => props.website,
  (newWebsite) => {
    if (newWebsite) {
      // 逐个字段赋值，避免未定义字段覆盖
      Object.keys(state).forEach((key) => {
        if (key in newWebsite) {
          // @ts-ignore
          state[key] = newWebsite[key];
        } else {
          // @ts-ignore
          state[key] = getDefaultState()[key];
        }
      });
    } else {
      Object.assign(state, getDefaultState());
    }
  },
  { immediate: true }
);

const resetForm = () => {
  Object.assign(state, getDefaultState());
};

const closeModal = () => {
  isOpen.value = false;
  resetForm();
};

async function onSubmit(event: FormSubmitEvent<Schema>) {
  console.log("tags:", state.tags);

  if (loading.value) return; // 防止重复提交
  loading.value = true;
  try {
    // 先校验数据
    const parsed = schema.parse(state);

    const url = "/api/websites";
    const isAdd = !!parsed.id;
    const method = isAdd ? "PUT" : "POST";

    const res = await $fetch(url, {
      method,
      body: parsed,
    });

    if (res.code === RESPONSE_STATUS_CODE.SUCCESS) {
      toast.add({
        title: isAdd ? "编辑成功" : "新增成功",
        color: "success",
        icon: "ri:checkbox-circle-line",
      });
      emit("success");
      closeModal();
    } else {
      toast.add({
        title: res.msg || "操作失败",
        color: "error",
        icon: "ri:close-circle-line",
      });
    }
  } catch (error) {
    // zod 校验错误或请求异常
    if (error?.issues) {
      // zod错误
      error.issues.forEach((issue: any) => {
        toast.add({
          title: issue.message,
          color: "error",
          icon: "ri:close-circle-line",
        });
      });
    } else {
      toast.add({
        title: "操作失败",
        color: "error",
        icon: "ri:close-circle-line",
      });
    }
  } finally {
    loading.value = false;
  }
}
</script>
