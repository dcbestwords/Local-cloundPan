<template>
  <div>
    <!-- 主视图 -->
    <ContextMenu :menu="mainMenu" @select="handleSelect">
      <div class="pan-wrapper">
        <h1 class="pan-title">我的云盘</h1>
        <!-- tabbar -->
        <div class="tabbar-container">
          <Tabbar
            @switch-mode="toggleView"
            :on-successed="() => getFileList(route.query.path as string)"></Tabbar>
        </div>

        <!-- 列表视图 -->
        <ContextMenu
          v-if="curMode === 0"
          :menu="fileMenu"
          class="list-view"
          showarea=".list-view .ep-table__row"
          @select="handleSelect">
          <el-table
            :data="list"
            row-key="id"
            ref="tableRef"
            show-overflow-tooltip
            max-height="calc(100vh - 12rem)"
            :row-style="{ height: '50px' }"
            v-loading="loading"
            @row-click="toggleCheck"
            @row-contextmenu="(row) => toggleCheck(row, true)"
            @selection-change="handleSelectionChange">
            <el-table-column fixed type="selection" width="55" reserve-selection />
            <el-table-column prop="name" label="文件名" min-width="100">
              <template #default="{ row }">
                <!-- 图标 -->
                <template v-if="row.fileType === 'picture'">
                  <el-image
                    class="file-pic"
                    fit="cover"
                    :src="row.thumbnailPath || row.filePath"
                    :lazy="true"
                    :preview-src-list="[row.filePath]"
                    preview-teleported
                    @click.stop="() => {}" />
                </template>
                <img v-else :src="getAssetsFile(row)" class="file-pic" />

                <!-- 文件名 -->
                <template v-if="row.isRename">
                  <input
                    ref="renameIpt"
                    type="text"
                    name="filename"
                    class="rename-ipt"
                    v-focus
                    :value="row.name"
                    @click.stop="() => {}" />
                  <ElButton
                    type="primary"
                    size="small"
                    class="rename-btn"
                    @click.stop="finishRenameOrCreate(row)">
                    <ElIcon size="14"><Select /></ElIcon>
                  </ElButton>
                  <ElButton
                    type="primary"
                    size="small"
                    class="rename-btn"
                    @click.stop="cancelRenameOrCreate(row)">
                    <ElIcon size="14"><CloseBold /></ElIcon>
                  </ElButton>
                </template>
                <span v-else class="filename" @click.stop="handleClick(row)">{{ row.name }}</span>
              </template>
            </el-table-column>
            <el-table-column prop="size" label="大小" min-width="30"></el-table-column>
            <el-table-column prop="modified" label="修改时间" min-width="70"></el-table-column>
          </el-table>
        </ContextMenu>

        <!-- 网格视图 -->
        <ContextMenu
          v-else
          class="grid-wrapper"
          :menu="fileMenu"
          showarea=".grid-view .file-item"
          @select="handleSelect">
          <el-scrollbar
            ref="gridScrollContainerRef"
            max-height="calc(100vh - 12rem)"
            :wrap-style="{ padding: '0 25px' }">
            <!-- v-bind="wrapperProps" -->
            <div class="grid-view" ref="gridViewWrapper">
              <GridView
                :file-list="list"
                @file-click="handleClick"
                @file-choosed="handleChoose"
                @finish="finishRenameOrCreate"
                @cancel="cancelRenameOrCreate" />
            </div>
          </el-scrollbar>
        </ContextMenu>

        <!-- 数量信息 -->
        <p class="pan-card-footer">{{ allDate.length }}个项目</p>
      </div>
    </ContextMenu>

    <!-- 移动目录选择框 -->
    <MoveSelect
      v-if="mvOrCopyShow"
      v-model:mvOrCopyShow="mvOrCopyShow"
      :filenameList="filenameList"
      :popoverType="mvOrCopy"
      :on-success="() => getFileList(route.query.path as string)" />

    <!-- 详情展示页 -->
    <PlayPage v-if="playPageShow" v-model:playPageShow="playPageShow" :playInfo="playInfo" />

    <!-- 下载 -->
    <div class="download-container">
      <a ref="downloadRef" :href="fileHref"></a>
    </div>
  </div>
</template>

<script setup lang="ts">
  import { ref, shallowRef, watch, computed, nextTick, onMounted } from 'vue';
  import { reqFileList, delteFile, renameFile, createDir } from '@/api/file/fileList';
  import { getAssetsFile } from '@/utils/tool';
  import { formatFile } from '@/api/file/types';
  import { ElMessage, ElMessageBox, ElScrollbar, ElTable } from 'element-plus';
  import { useRouter, useRoute } from 'vue-router';

  import { getFileType, formatFileSize, formatDateTime } from '@/utils/filestatus';
  import useVitualList from '@/composables/useVitualList';

  import ContextMenu from '@/components/ContextMenu/index.vue';
  import Tabbar from './components/Tabbar/index.vue';
  import MoveSelect from './components/MoveSelect/index.vue';
  import PlayPage from './components/PlayPage/index.vue';
  import GridView from './components/GridView/index.vue';

  defineOptions({ name: 'sharePan' });
  const router = useRouter();
  const route = useRoute();
  const selectedFiles = ref<formatFile[]>([]); // 已勾选的文件

  // #region 虚拟列表
  const allDate = shallowRef([]);
  const tableRef = ref<InstanceType<typeof ElTable>>(null);
  const gridScrollContainerRef = ref<InstanceType<typeof ElScrollbar>>(null);
  const gridViewWrapper = ref<HTMLElement | null>(null);
  let scrollContainerRef = ref(null);
  let wrapper = ref(null);
  const itemHeight = computed(() => (curMode.value === 0 ? 50 : 250));
  const curMode = ref<0 | 1>(0); // 0: 列表视图 1: 网格视图
  const { list } = useVitualList(allDate, {
    wrapper,
    itemHeight, // 元素高度
    scrollContainerRef, // 滚动容器
    flexItemWidth: 184, // 网格视图单个元素宽度
    curMode, // 当前视图模式
  });
  // #endregion

  // #region 准备文件数据
  const loading = ref(false);
  const getFileList = async (path: string, sortMode?: 'name' | 'size' | 'modified') => {
    loading.value = true;
    const result = await reqFileList(path, sortMode);
    if (result.code === 200) {
      const origin = window.location.origin;
      // 格式化数据
      const formatRes: formatFile[] = result.data.fileList.reduce((newList, file) => {
        let { ext, isDir, size, modified, filePath, thumbnailPath } = file;
        const fileType = getFileType(ext); // 文件类型
        size = formatFileSize(isDir, size as number);
        modified = formatDateTime(modified as number);
        filePath = new URL(filePath, origin).href;
        thumbnailPath = new URL(thumbnailPath, origin).href;
        newList.push({
          ...file,
          size,
          modified,
          fileType,
          filePath,
          thumbnailPath,
          isRename: false,
        });
        return newList;
      }, []);
      allDate.value = formatRes;
      clearSelection();
      resetScroll();
      nextTick(() => {
        loading.value = false;
      });
    }
  };
  getFileList(route.query.path as string);
  // #endregion

  // #region 右键菜单
  const mainMenu = [
    // 菜单项
    {
      label: '排序方式',
      subMenu: [{ label: '名称' }, { label: '大小' }, { label: '修改日期' }],
    },
    { label: '新建文件夹' },
    { label: '刷新' },
  ];
  const fileMenu = computed(() => {
    const baseMenu = [
      { label: '下载', icon: 'Download' },
      { label: '复制', icon: 'CopyDocument' },
      { label: '移动', icon: 'Rank' },
      { label: '删除', icon: 'Delete' },
    ];
    return selectedFiles.value.length > 1
      ? baseMenu
      : [...baseMenu, { label: '重命名', icon: 'Edit' }];
  });
  const mvOrCopyShow = ref(false);
  const playPageShow = ref(false);
  const mvOrCopy = ref<'move' | 'copy'>('move');
  const playInfo = ref({ name: '', url: '', type: '' });
  const downloadRef = ref<HTMLElement>(null);
  const handleSelect = (MenuItem) => {
    // 菜单点击事件
    const path = route.query.path as string;
    const handleDownload = () => {
      downloadRef.value.click();
      clearSelection();
    };
    const handleDelete = () => {
      ElMessageBox.confirm('文件一经删除，无法恢复，确定删除所选的文件吗？', '确定删除', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning',
      })
        .then(async () => {
          const result = await delteFile(path, filenameList.value);
          if (result.code !== 200) {
            ElMessage({
              type: 'warning',
              message: result.msg,
            });
          } else {
            ElMessage({
              type: 'success',
              message: '删除成功！',
            });
          }
          getFileList(path);
        })
        .catch(() => {
          ElMessage({
            type: 'info',
            message: '删除失败',
          });
        });
    };
    const handleRename = () => {
      allDate.value.forEach((file) => {
        file.isRename = false;
      });
      selectedFiles.value[0].isRename = true;
    };
    const handleMove = () => {
      mvOrCopy.value = 'move';
      mvOrCopyShow.value = true;
    };
    const handleCopy = () => {
      mvOrCopy.value = 'copy';
      mvOrCopyShow.value = true;
    };
    const handleCreateDir = () => {
      allDate.value.unshift({
        name: '新建文件夹',
        size: '-',
        isRename: true,
        isCreate: true,
        isDir: true,
        modified: formatDateTime(Date.now()),
      });
    };
    const operation = {
      '排序方式-名称': () => getFileList(path, 'name'),
      '排序方式-大小': () => getFileList(path, 'size'),
      '排序方式-修改日期': () => getFileList(path, 'modified'),
      刷新: () => getFileList(path),
      新建文件夹: handleCreateDir,
      下载: handleDownload,
      移动: handleMove,
      复制: handleCopy,
      重命名: handleRename,
      删除: handleDelete,
    };
    operation[MenuItem.label]();
  };
  // #endregion

  // 已勾选的文件名列表
  const filenameList = computed(() => {
    return selectedFiles.value.map((file) => file.name);
  });
  // 文件的下载链接
  const fileHref = computed(() => {
    return new URL(
      `api/download?filenameList=[${filenameList.value}]&path=${route.query.path ? route.query.path : ''}`,
      window.location.origin
    ).href;
  });

  // #region 两种mode公共事件
  // 1. 文件点击
  const handleClick = (row: formatFile) => {
    const showPlayWindow = (name: string, url: string, type: string) => {
      playPageShow.value = true;
      playInfo.value = { name, url, type };
    };
    const queryParam = route.query.path === '' ? '' : route.query.path + '/';
    // 目录
    if (row.isDir) {
      router.push({ query: { path: `${queryParam}${row.name}` } });
    }
    // 音视频、pdf、txt、md
    else if (row.fileType === 'video' || row.fileType === 'audio') {
      const { fileType: type, filePath: url, name } = row;
      showPlayWindow(name, url, type);
    } else if (row.fileType === 'document' && ['pdf', 'txt', 'md'].includes(row.ext)) {
      const { ext: type, filePath: url, name } = row;
      showPlayWindow(name, url, type);
    } else if (row.fileType === 'picture') {
      ElMessage({
        type: 'info',
        message: '请点击图片查看原图！',
      });
    } else {
      ElMessage({
        type: 'warning',
        message: '此类文件无法预览，请下载后查看！',
      });
    }
  };
  // 2. 文件重命名或创建
  const renameIpt = ref<HTMLInputElement>(null);
  const finishRenameOrCreate = async (row, name?: string) => {
    const newName = renameIpt.value?.value || name;
    const operateText = row.isCreate ? '新建文件夹' : '重命名';
    // 文件名不为空
    if (!newName) {
      ElMessage({
        type: 'warning',
        message: '文件名不能为空！',
      });
      return;
    }
    // 新旧文件名不能相同
    if (!row.isCreate && newName === row.name) {
      row.isRename = false;
      ElMessage({
        type: 'warning',
        message: '新文件名不能与旧文件名相同！',
      });
      return;
    }
    let result = null;
    const handleCreateDir = async () => {
      const isExist = allDate.value.slice(1).some((f) => f.name === newName);
      if (isExist) {
        try {
          await ElMessageBox.confirm(
            `此位置已包含同名文件夹，继续操作将替换旧文件夹的内容，是否进行替换？`,
            '确定替换',
            {
              confirmButtonText: '确定',
              cancelButtonText: '取消',
              type: 'warning',
            }
          );
          result = await createDir(route.query.path as string, newName);
        } catch {
          ElMessage({
            type: 'info',
            message: '取消创建',
          });
          cancelRenameOrCreate(row);
          return; // 退出函数
        }
      } else {
        result = await createDir(route.query.path as string, newName);
      }
    };
    const handleRenameFile = async () => {
      result = await renameFile(route.query.path as string, row.name, newName);
    };
    if (row.isCreate) {
      await handleCreateDir();
      delete row.isCreate;
    } else {
      await handleRenameFile();
    }

    if (result && result.code === 200) {
      ElMessage({
        type: 'success',
        message: `${operateText}成功！`,
      });
      row.isRename = false;
      await getFileList(route.query.path as string);
    }
  };
  // 3. 取消重命名或创建
  const cancelRenameOrCreate = (row) => {
    if (row.isCreate) {
      allDate.value.shift();
    }
    row.isRename = false;
    tableRef.value?.toggleRowSelection(row, false);
  };
  // 4. 重置滚动条位置
  const resetScroll = () => {
    tableRef?.value?.setScrollTop(0);
    gridScrollContainerRef?.value?.setScrollTop(0);
  };
  // #endregion

  // #region 网格专属事件
  // 勾选文件
  const handleChoose = (checkedList) => {
    selectedFiles.value = allDate.value.filter((file) => checkedList[file.id]);
  };
  // #endregion

  // #region 列表专属事件
  // 文件勾选
  const handleSelectionChange = (select) => {
    selectedFiles.value = select;
  };
  // 清除所有勾选状态
  const clearSelection = () => {
    selectedFiles.value = [];
    tableRef?.value?.clearSelection();
  };
  // 切换勾选状态
  const toggleCheck = (row, isSingle?: boolean) => {
    // 单选（鼠标右键）
    if (isSingle === true) {
      if (selectedFiles.value.findIndex((file) => file.id === row.id) !== -1) {
        return;
      }
      tableRef.value.clearSelection();
      tableRef.value.setCurrentRow(row);
    }
    // 切换勾选状态（鼠标左键）
    // @ts-ignore
    tableRef.value.toggleRowSelection(row);
  };
  // #endregion

  // 视图切换
  const toggleView = (mode: 0 | 1) => {
    curMode.value = mode;
    selectedFiles.value = [];
  };

  watch(
    () => route.query.path,
    (newPath) => {
      getFileList(newPath as string);
    }
  );

  // #region 变化ScrollContainerRef的值
  const getScrollContainer = () => {
    if (curMode.value === 0) {
      // @ts-ignore
      scrollContainerRef.value = tableRef.value.$refs.scrollBarRef.wrapRef;
      wrapper.value = (scrollContainerRef.value as HTMLElement).querySelector(
        '.ep-scrollbar__view'
      );
    } else {
      scrollContainerRef.value = gridScrollContainerRef.value?.wrapRef;
      wrapper.value = gridViewWrapper.value;
    }
  };
  watch(curMode, getScrollContainer, { flush: 'post' });
  onMounted(getScrollContainer);
  // #endregion
</script>

<style scoped lang="scss">
  .pan-wrapper {
    border-radius: 10px;
    max-width: 100%;
    min-width: 43rem;
    max-height: calc(100vh - 4rem);
    .pan-title {
      font-size: 1.5rem;
      line-height: 2rem;
      font-weight: 700;
      text-align: left;
      margin-bottom: 1.5rem;
    }
    .tabbar-container {
      margin-bottom: 1rem;
    }
    .filename {
      transition: all 0.3s ease;
      cursor: pointer;
      &:hover {
        color: var(--ep-menu-active-color);
      }
    }
    .list-view {
      .file-pic {
        width: 1.5rem;
        height: 1.5rem;
        vertical-align: bottom;
        margin-right: 0.3rem;
      }
      .rename-ipt {
        width: 40%;
        margin-right: 0.75rem;
        line-height: 1.4rem;
        height: 1.4rem;
        border-radius: 4px;
        border: 1px solid #a7adbc;
        padding: 0 0.6rem;
        outline: none;
        font-size: 0.8125rem;
      }
      .rename-btn {
        width: 1.25rem;
        height: 1.25rem;
        padding: 2px;
      }
    }
    .grid-view {
      display: flex;
      flex-wrap: wrap;
      height: fit-content;
      align-content: start;
    }
    .pan-card-footer {
      font-size: 1rem;
      line-height: 1rem;
      text-align: left;
      margin-top: 0.8rem;
    }
  }
  // :deep(.ep-table__body tr:hover > td.ep-table__cell) {
  //   transform: scale(1.01);
  // }
  // :deep(.ep-table__body tr > td.ep-table__cell) {
  //   transition: all 0.3s ease;
  // }
  .download-container {
    position: absolute;
    left: -9999px;
  }
</style>
