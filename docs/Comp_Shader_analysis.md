# 简要分析3DGS.CPP项目中的各个Compute task的职责
## 1. precomp_cov3d.comp
precomp_cov3d.comp 做的是离线（场景加载时）预计算每个高斯的 3D 协方差矩阵，并把结果写进 cov3ds buffer，供后续每帧 preprocess.comp 使用。
核心流程是：
- 从 Vertex 里读每个高斯的 scale_opacity.xyz 和 rotation（四元数）。
- 用 scale_factor 缩放尺度，构造对角缩放矩阵 S。
- 把四元数转成旋转矩阵 R。
- 令 M = S * R，再算 cov3d = transpose(M) * M（保证对称半正定）。
- 只存 6 个独立分量（对称矩阵上三角）：xx, xy, xz, yy, yz, zz，减少带宽和存储。
  
例如这里就是关键计算与写回：
```glsl
mat3 S = mat3(1.0);
S[0][0] = vertices[index].scale_opacity.x * scale_factor;
S[1][1] = vertices[index].scale_opacity.y * scale_factor;
S[2][2] = vertices[index].scale_opacity.z * scale_factor;

// Compute rotation matrix from quaternion
mat3 R = rotationFromQuaternion(vertices[index].rotation);

mat3 M = S * R;
mat3 cov3d = transpose(M) * M;

cov3ds[index * 6] = cov3d[0][0];
cov3ds[index * 6 + 1] = cov3d[0][1];
cov3ds[index * 6 + 2] = cov3d[0][2];
cov3ds[index * 6 + 3] = cov3d[1][1];
cov3ds[index * 6 + 4] = cov3d[1][2];
cov3ds[index * 6 + 5] = cov3d[2][2];
```
它在整个渲染流程中的角色：
- **时机**：在场景加载时调用一次，不是每帧都算。GSScene::load() 里会执行 precomputeCov3D()，dispatch 这个 shader。
- **定位**：这是“几何先验预计算”阶段，位于每帧 preprocess -> prefix_sum -> sort -> tile_boundary -> render 之前。
- **作用**：把相机无关的 3D 形状信息（高斯椭球）提前算好；每帧 preprocess.comp 只需结合当前相机把 3D 协方差投影成 2D 协方差，再算屏幕空间 conic/radius/AABB。
- **价值**：避免每帧重复做四元数旋转与 3D 协方差构造，降低每帧计算量。
  
preprocess.comp 里就是直接读取这个预计算结果并重建 Sigma：
```glsl
mat3 Sigma = mat3(
    cov3ds[index * 6], cov3ds[index * 6 + 1], cov3ds[index * 6 + 2],
    cov3ds[index * 6 + 1], cov3ds[index * 6 + 3], cov3ds[index * 6 + 4],
    cov3ds[index * 6 + 2], cov3ds[index * 6 + 4], cov3ds[index * 6 + 5]
);
mat3 T = W * J;
mat3 cov2d = transpose(T) * Sigma * T;
```
一句话总结：precomp_cov3d.comp 是把“每个高斯在世界空间的椭球形状”预先固化成紧凑协方差数据，作为后续屏幕空间 splatting 的基础输入。

---

## 2. preprocess.comp
preprocess.comp 是每帧渲染里的 **高斯屏幕空间预处理**阶段。
它把场景里的每个 3D Gaussian 从“世界/相机空间参数”转换成“后续排序与合成可直接使用的 2D 参数”。
对每个高斯（每个 invocation 对应一个 index）主要做这些事：
- 初始化并做早期剔除
- - 先把输出默认清零（radii=0、tiles_overlap=0）
- - 投影到视图空间后，如果 z <= 0.2 直接剔除
- 把 3D 协方差投影到 2D
- - 读取 precomp_cov3d.comp 预计算的 cov3ds
- - 结合当前相机的 Jacobian 与 view 旋转，得到屏幕空间 cov2d
- - det <= 0 的异常情形直接剔除
- 生成 splatting 所需的 conic + opacity
- - 对 cov2d 求逆，得到二次型参数（conic）
- - 写入 attr[index].conic_opacity
- 估计屏幕半径与覆盖瓦片范围
- - 通过特征值估计高斯半径 radii = ceil(3 * sqrt(lambda_max))
- - 计算中心像素坐标 uv
- - 生成 tile 级别 AABB（覆盖哪些 tile）
- - 写入 tiles_overlap[index]（给 prefix-sum 统计实例数）
- 计算颜色与深度等属性
- - 用 SH 系数 + 视线方向算颜色
- - 写入 depth, uv, color, radii, aabb, magic

关键片段可以看到它把几何参数“压成”后续管线可用数据：
```glsl
mat2 cov2d = compute_cov2d(p_view.xyz);
float det = determinant(cov2d);
if (det <= 0.0) {
    return;
}
mat2 conic = inverse(cov2d);
attr[index].conic_opacity.xyz = vec3(conic[0][0], conic[0][1], conic[1][1]);
attr[index].conic_opacity.w = vertices[index].scale_opacity.w;

float mid = 0.5 * (cov2d[0][0] + cov2d[1][1]);
float lambda1 = mid + sqrt(max(0.1, mid * mid - det));
float lambda2 = mid - sqrt(max(0.1, mid * mid - det));
float lambda = max(lambda1, lambda2);
float radii = ceil(3.0 * sqrt(lambda));

vec2 uv = vec2(ndc2Pix(ndc.x, int(width)), ndc2Pix(ndc.y, int(height)));
...
tiles_overlap[index] = num_tiles_overlap;
attr[index].depth = p_view.z;
attr[index].color_radii.w = radii;
attr[index].color_radii.xyz = compute_sh();
attr[index].uv = uv;
```
**在整个渲染流程中的角色**
它是每帧第一步核心计算（在 Renderer::recordPreprocessCommandBuffer() 开头先 dispatch 它），作用是为后续阶段准备输入：

- 给 prefix_sum：tiles_overlap[]（每个高斯覆盖多少 tile）
- 给 preprocess_sort / sort：attr[] 里的 depth、aabb、uv 等用于生成 key/value 与排序
- 给最终 render.comp：conic_opacity + color + uv 直接进行每像素高斯混合
可以把它理解为：
把“3D Gaussian + 相机参数”变成“2D raster/splat 参数”的转换层。没有这一步，后面的排序、tile 边界构建、像素合成都无法高效进行。

---

## 3. prefix_sum.comp
**prefix_sum.comp 在算什么？**
它在一个长度为 N = vertices.length() 的数组上做并行前缀和（扫描），输入是每个高斯覆盖的 tile 数量 tiles_overlap[i]（由 preprocess.comp 写入 tileOverlapBuffer，再拷到 prefixSumPingBuffer）。
每一轮用 push constant timestep 控制步长 (2^{\texttt{timestep}})，在 src / dst 两个 buffer 之间乒乓：偶数步把结果写到 dst，奇数步写回 src，避免同一步里读写冲突。核心更新是：
```glsl
    if (timestep % 2 == 0) {
        if (index < pow(2, timestep)) {
            dst[index] = src[index];
        } else {
            uint index2 = index - power(2, timestep);
            dst[index] = src[index] + src[index2];
        }
    } else {
        if (index < pow(2, timestep)) {
            src[index] = dst[index];
        } else {
            uint index2 = index - power(2, timestep);
            src[index] = dst[index] + dst[index2];
        }
    }
```
**在整条渲染链路里干什么？**
1. **预处理（recordPreprocessCommandBuffer）**：preprocess 算每个可见高斯的 tile 包围盒，并把与该高斯相交的 tile 个数写入 tiles_overlap。
2. **前缀和**：prefix_sum 对上述 tiles_overlap 做扫描；CPU 只把最后一个元素拷到 totalSumBufferHost，得到本帧需要的总实例数
(\sum_i \texttt{tiles_overlap}[i])（每个「高斯 × 覆盖到的 tile」对应排序/绘制里的一条记录）。
3. **展开排序键（preprocess_sort.comp）**：用前缀和数组给每个高斯分配在 keys/payloads 里的一段连续区间——起始下标是「前一个前缀和」，结束用断言对齐到当前前缀和：
   ```glsl
       uint ind = index == 0 ? 0 : prefixSum[index - 1];
    ...
            keys[ind] = k;
            payloads[ind] = index;
            ind++;
    ...
    assert(ind == prefixSum[index], "ind: %d", ind);
   ```
4. 之后才是 radix sort 等，按 (tile, depth) 排序，再交给光栅/合成。

**一句话**：prefix_sum.comp 不负责画像素，它把「每个高斯占多少条 (tile, 高斯) 记录」变成前缀和，既给出总条数（分配/扩容 sort buffer），又给 preprocess_sort 提供无冲突的写入区间，是 3DGS 里「按 tile 展开实例 → 再排序」的桥梁。

---

## 4. RadixSortPipeline
### hist.comp：按当前 8 位数字做「分桶计数」
作用：对当前这一趟 radix（由 g_shift 指定取哪 8 位）统计直方图。

- 每个 workgroup（256 线程）用共享内存里的 histogram[256]，对分配给本组的一段元素循环：
- - bin = (key >> g_shift) & 255
- - atomicAdd(histogram[bin], 1)
- barrier() 后，线程 0..255 把本组直方图写到全局：
g_histograms[RADIX_SORT_BINS * wID + lID]。

也就是说：它不排序，只回答「每个桶、在每个 workgroup 里有多少个元素」，供下一步计算全局偏移。
```glsl
for (uint index = 0; index < g_num_blocks_per_workgroup; index++) {
    uint elementId = wID * g_num_blocks_per_workgroup * WORKGROUP_SIZE + index * WORKGROUP_SIZE+ lID;
    if (elementId < g_num_elements) {
        // determine the bin
        const uint bin = uint(g_elements_in[elementId] >> g_shift) & (RADIX_SORT_BINS - 1);
        // increment the histogram
        atomicAdd(histogram[bin], 1U);
    }
}
barrier();
if (lID < RADIX_SORT_BINS) {
    g_histograms[RADIX_SORT_BINS * wID + lID] = histogram[lID];
}
```
### sort.comp：根据直方图算偏移并 scatter（稳定重排）
分两阶段（同一 shader 内）：

1. 全局前缀和 / 每个 workgroup 在全局中的起始位置（约 110–131 行）
用 subgroup 在桶维度上把各 workgroup 的 g_histograms 累加，得到每个 bin 的全局 exclusive 型偏移，并配合 local_histogram 得到本 workgroup 在处理该 bin 时的写入基址 global_offsets[binID]。

2. 按桶 scatter 键和载荷（约 133–212 行）
对每个元素再算 bin，用 原子 OR 到 bit 图 + bitCount 算在本组同 bin 内的局部排名，最终写入
g_elements_out[binOffset + prefix]、g_payload_out[binOffset + prefix]。
这是 稳定的 一轮 radix，保证同一桶内相对顺序与输入一致。

因此：sort.comp = 本趟 radix 的「排好序的一遍」，输入/输出在不同 ping-pong 缓冲之间切换。

**在整条渲染里扮演什么角色?**
数据流大致是：

1. 预处理 + 前缀和：得到实例总数 numInstances。
2. preprocess_sort：生成未排序的 (tile, depth) 键和 Gaussian id 载荷。
3. 基数排序（hist + sort × 8）：把 numInstances 条记录按 64 位键 全局排序。键的高位是 tile、低位是深度相关的 floatBitsToUint，因此排序后 同一 tile 的条目聚在一起，且在同一 tile 内按深度键有序，便于后续按 tile 做边界与合成。
4. tile_boundary：在排好序的键上找每个 tile 的范围。
5. render：按 tile 绘制高斯。

所以：**hist.comp** 是每一趟 radix 的「计数」；**sort.comp** 是每一趟的「根据计数算位置并写回」。两者合在一起才完成 GPU 上的 radix sort，这是 3DGS 里 「实例列表 → 按 tile/深度有序 → 再光栅」 的核心步骤；没有这一步，就无法高效按 tile 批处理并保持正确的深度顺序。

---

## 5. preprocess_sort.comp
### preprocess_sort.comp 具体在做什么？
它按高斯索引 index 并行执行，只对在预处理里被判定为可见的高斯生效（attr[index].color_radii.w != 0，与 preprocess.comp 里剔除逻辑一致）。
对每个这样的高斯，它做三件事：
1. 用前缀和拿写入区间
prefixSum 来自前面 prefix_sum 对 tiles_overlap 的包含前缀和。
本高斯在 keys / payloads 里的起始下标是「前面所有高斯的 tile 实例数之和」：
```glsl
    uint ind = index == 0 ? 0 : prefixSum[index - 1];
```
2. 遍历屏幕 AABB 里每个 tile
   双重循环 i ∈ [aabb.x, aabb.z)、j ∈ [aabb.y, aabb.w)，与 preprocess.comp 里写的 tile 包围盒一致。
3. 为每个 (高斯, tile) 写一条排序记录
   **键**：tileIndex（列主或按 tileX 展开的线性 tile 下标）放在高 32 位，深度用 floatBitsToUint(depth) 放在低 32 位，拼成 64 位，供后续 radix sort：
   ```glsl
       for (uint i = attr[index].aabb.x; i < attr[index].aabb.z; i++) {
        for (uint j = attr[index].aabb.y; j < attr[index].aabb.w; j++) {
            uint64_t tileIndex = i + j * tileX;
            uint depthBits = floatBitsToUint(attr[index].depth);
            uint64_t k = (tileIndex << 32) | uint64_t(depthBits);
            keys[ind] = k;
            payloads[ind] = index;
            ind++;
        }
    }
   ```
   **载荷**: payloads[ind] = index，即高斯在场景里的顶点索引（同一高斯在多个 tile 上会出现多条记录，载荷相同）。
   tileX 由 CPU 经 push constant 传入（每行 tile 个数），与屏幕分辨率、tile 尺寸一致。

### 在整个渲染里扮演什么角色？
可以把整条链路想成：
| 阶段 | 作用 | 备注 |
| :--- | :---: | ---: |
| preprocess | 每个高斯：视锥、2D 协方差、tile AABB、深度、颜色等；tiles_overlap = 覆盖 tile 个数。 |  |
| prefix_sum | 对 tiles_overlap 做前缀和，得到总实例数并为每个高斯分配紧凑区间。 |  |
| preprocess_sort | 把「每高斯一条」展开成「每 (高斯, tile) 一条」，填入 sortKBufferEven / sortVBufferEven。 |  |
| hist + sort | 按 64 位键排序（先按 tile、再按深度位型）。 |  |
| tile_boundary / render | 按排好序的列表按 tile 绘制。。 |  |
	
因此 preprocess_sort.comp 的角色是：从「按高斯存储的属性 + 前缀和」生成「未排序的扁平实例表」——表里每一项对应「这个高斯要贡献到哪一个 tile」，键用于后续排序，载荷告诉光栅化阶段是哪个高斯。没有这一步，后面的 radix sort 和按 tile 渲染就没有输入数据。

---

## 6. tile_boundary.comp
### tile_boundary.comp 在算什么？
输入是 radix 排好序的 uint64_t keys[]（与 preprocess_sort / sort 管线里一致：高 32 位是线性 tile 下标 tileIndex，低 32 位是深度位型）。
输出 boundaries[] 长度是 tileCount × 2：对每个 tile 编号 key 存一对 [start, end) 在排序列表里的下标范围。

每个线程处理排序数组里的一个下标 index（0 .. numInstances-1），取出当前与上一个元素的 tile 键（高 32 位）：
```glsl
    uint key = uint(keys[index] >> 32);
    if (index == 0) {
        boundaries[key * 2] = index;
    } else {
        uint prevKey = uint(keys[index - 1] >> 32);
        ...
        if (key != prevKey) {
            boundaries[key * 2] = index;
            boundaries[prevKey * 2 + 1] = index;
```
含义简要如下：
| 情况 | 写入 | 备注 |
| :--- | :---: | ---: |
| index == 0 | 第一个 tile 的 起始 boundaries[key*2] = 0 |  |
| key != prevKey（tile 变了） | 新 tile 的 起始 boundaries[key*2] = index；旧 tile 的 结束（开区间） boundaries[prevKey*2+1] = index |  |
| index == numInstances - 1 | 最后一个 tile 的 结束 boundaries[key*2+1] = numInstances |  |

也就是说：在已按 tile（再按深度）排好序的一维数组上，扫一遍 tile 边界，为每个 tile 记下 sorted 里 从哪一项开始、到哪一项结束（不含）。

Renderer 里会先 fillBuffer(..., 0) 清空 tileBoundaryBuffer，再跑本 shader；没有 splat 的 tile 不会被写入，保持 0,0，渲染时 for (i=start; i<end) 自然循环 0 次。

### 在整个渲染里扮演什么角色？
数据流位置：
preprocess → prefix_sum → preprocess_sort → radix sort（hist/sort）→ tile_boundary → render

render.comp 里每个屏幕 tile 用同一套 tileX, tileY → 线性 tile 下标**去查这对边界，再在 sorted_vertices` 里只遍历 属于本 tile 的那一段（按深度顺序叠加高斯）：
```glsl
    uint tiles_width = ((width + TILE_WIDTH - 1) / TILE_WIDTH);

    uint start = boundaries[(tileX + tileY * tiles_width) * 2];
    uint end = boundaries[(tileX + tileY * tiles_width) * 2 + 1];
    ...
    for (uint i = start; i < end; i++) {
        uint vertex_key = sorted_vertices[i];
```

因此 tile_boundary.comp 的角色是：把「全局一条、已排序的 splat 列表」变成「每个 tile 在列表里的子区间」，让 render 不必在全数组上筛选，只对当前 tile 做 O(该 tile 上 splat 数) 的融合，这是 tiled 3DGS 里连接 排序结果 和 按 tile 光栅 的索引表。

---

## 7. render.comp
### 它具体做了什么?
每个 workgroup 对应一个 tile，local_size = TILE_WIDTH × TILE_HEIGHT，所以每个线程对应 tile 内一个像素：
```glsl
void main() {
    uint tileX = gl_WorkGroupID.x;
    uint tileY = gl_WorkGroupID.y;
    ...
    uvec2 curr_uv = uvec2(tileX * TILE_WIDTH + localX, tileY * TILE_HEIGHT + localY);
    ...
    uint start = boundaries[(tileX + tileY * tiles_width) * 2];
    uint end = boundaries[(tileX + tileY * tiles_width) * 2 + 1];
```
然后对这个 tile 的排序结果区间 [start, end) 做前向 alpha 合成：
1. 取当前记录对应高斯 vertex_key = sorted_vertices[i]
2. 读该高斯屏幕中心 uv、二次型参数 conic_opacity、颜色 color_radii.xyz
3. 用高斯椭圆公式算该像素上的权重（power）
4. 得到不透明度 alpha = min(0.99, opacity * exp(power))
5. 累积颜色并更新透射率 T
```glsl
for (uint i = start; i < end; i++) {
    uint vertex_key = sorted_vertices[i];
    ...
    float power = -0.5f * (...) - ...;
    if (power > 0.0f) continue;

    float alpha = min(0.99f, co.w * exp(power));
    if (alpha < 1.0f / 255.0f) continue;

    float test_T = T * (1 - alpha);
    if (test_T < 0.0001f) break;

    c += attr[vertex_key].color_radii.xyz * alpha * T;
    T = test_T;
}
```
最后写入图像：
```glsl
imageStore(output_image, ivec2(curr_uv), vec4(c, 1.0f));
```

### 在整个渲染流程里的角色
它是终端着色/合成阶段，依赖前面所有步骤准备的数据：
- preprocess：算每个高斯的屏幕属性（uv、conic_opacity、颜色等）
- prefix_sum + preprocess_sort：把高斯展开成 (tile, gaussian) 记录
- hist + sort：按 (tile, depth) 排序
- tile_boundary：给每个 tile 生成 [start,end) 区间
- render.comp：按该区间逐像素融合，输出最终图像
一句话：前面都在“组织数据并排序”，render.comp 才是“真正落盘到像素”的那一层。

