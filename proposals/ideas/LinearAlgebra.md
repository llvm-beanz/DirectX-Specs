---
title: "0035 - Linear Algebra Matrix"
draft: true
params:
  authors:
    - llvm-beanz: Chris Bieneman
    - mapodaca-nv: Mike Apodaca
    - V-FEXrt: Ashley Coleman
    - damyanp: Damyan Pepper
    - pow2clk: Gregory Roth
    - tex3d: Tex Riddell
    - hekota: Helena Kotas
    - jenatali: Jesse Natalie
    - anupamachandra: Anupama Chandrasekhar
    - mjbedy: Michael Bedy
  sponsors:
    - tbd: TBD
  status: Draft
---

* Planned Version: SM 6.10

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHOULD**, **SHOULD NOT**,
and **MAY** describe normative requirements. Lowercase uses of these words are
informative.

## Introduction

GPUs are exceptional parallel data processors, but increasingly it is becoming
important to model operations with cross-thread data dependencies. In HLSL and
Direct3D these operations have been called Wave operations, or in Vulkan
Subgroup operations. Related terms like Quad or derivatives have similar
meaning in different scoping contexts. Vulkan has also recently introduced the
term "cooperative" when talking about operations that require participation from
multiple threads, these can be viewed much like derivative operations but across
the full SIMD unit instead of a subset of threads.

All of these terms describe how the underlying instructions execute rather than
the operations they perform. This proposal instead describes those operations
as linear algebra.

## Motivation

HLSL has a Vulkan extension for SIMD matrix types [0021 - vk::Cooperative
Matrix](https://github.com/microsoft/hlsl-specs/blob/main/proposals/0021-vk-coop-matrix.md), and DirectX had previewed a similar feature in
SM 6.8 called [Wave Matrix](https://github.com/microsoft/hlsl-specs/pull/61).
This proposal is aimed at merging the two into a unified language feature that
can be supported on all platforms (with some platform-specific limitations).

This proposal is similar but not directly aligned with [0031 - HLSL Vector
Matrix Operations](https://github.com/microsoft/hlsl-specs/blob/main/proposals/0031-hlsl-vector-matrix-operations.md).

## Proposed solution

The following informative pseudo-HLSL API summarizes the proposed surface. The
detailed declarations and requirements in [HLSL API Documentation](#hlsl-api-documentation)
are normative and take precedence if this summary differs from them. The
summary uses `hlsl::enable_if` to represent template constraints.

Some portion of this API surface is portable between DirectX and Vulkan using the [proposed
DXIL](#dxil-operations) for DirectX and
[SPV_KHR_cooperative_matrix](https://github.com/KhronosGroup/SPIRV-Registry/blob/main/extensions/KHR/SPV_KHR_cooperative_matrix.asciidoc)
for Vulkan. Not all features proposed here are supported in Vulkan, so the API
as described is in the `dx` namespace.

A subsequent revision to HLSL's [0021 - Vulkan Cooperative
Matrix](https://github.com/microsoft/hlsl-specs/blob/main/proposals/0021-vk-coop-matrix.md) support could be considered separately to align
on a base set of functionality for inclusion in the `hlsl` namespace.

```c++
namespace dx {
namespace linalg {

template <ComponentEnum ElementType, uint DimA> struct VectorRef {
  ByteAddressBuffer Buf;
  uint Offset;
};

template <typename T, int N, ComponentEnum DT> struct InterpretedVector {
  vector<T, N> Data;
  static const ComponentEnum Interpretation = DT;
  static const SIZE_TYPE Size =
      __detail::ComponentTypeTraits<DT>::ElementsPerScalar * N;
};

template <ComponentEnum DT, typename T, int N>
InterpretedVector<T, N, DT> MakeInterpretedVector(vector<T, N> Vec) {
  InterpretedVector<T, N, DT> IV = {Vec};
  return IV;
}

template <ComponentEnum DestTy, ComponentEnum OriginTy, typename T, int N>
typename hlsl::enable_if<
    DestTy != OriginTy,
    InterpretedVector<typename __detail::ComponentTypeTraits<DestTy>::Type,
                      __detail::DstN<DestTy, OriginTy, N>::Value,
                      DestTy> >::type
Convert(vector<T, N> Vec) {
  vector<typename __detail::ComponentTypeTraits<DestTy>::Type,
         __detail::DstN<DestTy, OriginTy, N>::Value>
      Result;
  /* Do conversion somehow... */
  return MakeInterpretedVector<DestTy>(Result);
}

template <ComponentEnum DestTy, ComponentEnum OriginTy, typename T, int N>
typename hlsl::enable_if<DestTy == OriginTy,
                         InterpretedVector<T, N, DestTy> >::type
Convert(vector<T, N> Vec) {
  return MakeInterpretedVector<DestTy>(Vec);
}

template <ComponentEnum ComponentTy, SIZE_TYPE M, SIZE_TYPE N,
          MatrixUseEnum Use, MatrixScopeEnum Scope>
class Matrix {
  using ElementType = typename __detail::ComponentTypeTraits<ComponentTy>::Type;
  // If this isn't a native scalar, we have an 8-bit type, so we have 4 elements
  // packed in each scalar value.
  static const uint ElementsPerScalar =
      __detail::ComponentTypeTraits<ComponentTy>::ElementsPerScalar;
  static const bool IsNativeScalar =
      __detail::ComponentTypeTraits<ComponentTy>::IsNativeScalar;

  template <ComponentEnum NewCompTy, MatrixUseEnum NewUse = Use,
            bool Transpose = false>
  Matrix<NewCompTy, __detail::DimMN<M, N, Transpose>::M,
         __detail::DimMN<M, N, Transpose>::N, NewUse, Scope>
  Cast();

  template <typename T>
  [[nodiscard]] static
      typename hlsl::enable_if<hlsl::is_arithmetic<T>::value, Matrix>::type
      Splat(T Val);

  template <uint Align = __detail::DefaultAlign<ComponentTy, M, N>::Value>
  static Matrix Load(ByteAddressBuffer Res, uint StartOffset, uint Stride,
                     MatrixLayoutEnum Layout);

  template <uint Align = __detail::DefaultAlign<ComponentTy, M, N>::Value>
  static Matrix Load(RWByteAddressBuffer Res, uint StartOffset, uint Stride,
                     MatrixLayoutEnum Layout);

  template <typename T, SIZE_TYPE Size>
  static typename hlsl::enable_if<
      (hlsl::is_same<typename hlsl::strip_vector_type<T>::type,
                     ElementType>::value ||
       hlsl::is_integral_or_packed_integral<
           typename hlsl::strip_vector_type<T>::type>::value),
      Matrix>::type
  Load(groupshared T Arr[Size], uint StartIdx, uint Stride,
       MatrixLayoutEnum Layout);

  template <ComponentEnum LocalComp = ComponentTy>
  typename hlsl::enable_if<LocalComp == ComponentTy && IsNativeScalar,
                           uint>::type
  Length();

  template <ComponentEnum LocalComp = ComponentTy>
  typename hlsl::enable_if<LocalComp == ComponentTy && IsNativeScalar,
                           uint2>::type
  GetCoordinate(uint Index);

  template <ComponentEnum LocalComp = ComponentTy>
  typename hlsl::enable_if<LocalComp == ComponentTy && IsNativeScalar,
                           ElementType>::type
  Get(uint Index);

  template <ComponentEnum LocalComp = ComponentTy>
  typename hlsl::enable_if<LocalComp == ComponentTy && IsNativeScalar,
                           void>::type
  Set(uint Index, ElementType Value);

  template <uint Align = __detail::DefaultAlign<ComponentTy, M, N>::Value>
  void Store(RWByteAddressBuffer Res, uint StartOffset, uint Stride,
             MatrixLayoutEnum Layout);

  template <typename T, SIZE_TYPE Size>
  typename hlsl::enable_if<
      (hlsl::is_same<typename hlsl::strip_vector_type<T>::type,
                     ElementType>::value ||
       hlsl::is_integral_or_packed_integral<
           typename hlsl::strip_vector_type<T>::type>::value),
      void>::type
  Store(groupshared T Arr[Size], uint StartIdx, uint Stride,
        MatrixLayoutEnum Layout);

  // Accumulate methods
  template <uint Align = __detail::DefaultAlign<ComponentTy, M, N>::Value,
            MatrixUseEnum UseLocal = Use>
  typename hlsl::enable_if<Use == MatrixUse::Accumulator && UseLocal == Use,
                           void>::type
  InterlockedAccumulate(RWByteAddressBuffer Res, uint StartOffset, uint Stride,
                        MatrixLayoutEnum Layout);

  template <typename T, MatrixUseEnum UseLocal = Use, SIZE_TYPE Size>
  typename hlsl::enable_if<
      hlsl::is_arithmetic_vector<T>::value && Use == MatrixUse::Accumulator &&
          UseLocal == Use,
      void>::type
  InterlockedAccumulate(groupshared T Arr[Size], uint StartIdx, uint Stride,
                        MatrixLayoutEnum Layout);

#ifdef __hlsl_dx_compiler
  template <typename T, MatrixUseEnum UseLocal = Use, SIZE_TYPE Size>
  typename hlsl::enable_if<
      hlsl::is_same<typename hlsl::strip_vector_type<T>::type,
                    uint8_t4_packed>::value &&
          Use == MatrixUse::Accumulator && UseLocal == Use,
      void>::type
  InterlockedAccumulate(groupshared T Arr[Size], uint StartIdx, uint Stride,
                        MatrixLayoutEnum Layout);
#endif

  template <ComponentEnum CompTy, MatrixUseEnum UseLocal = Use>
  typename hlsl::enable_if<Use == MatrixUse::Accumulator && UseLocal == Use,
                           void>::type
  Accumulate(const Matrix<CompTy, M, N, MatrixUse::A, Scope> MatrixA);

  template <ComponentEnum CompTy, MatrixUseEnum UseLocal = Use>
  typename hlsl::enable_if<Use == MatrixUse::Accumulator && UseLocal == Use,
                           void>::type
  Accumulate(const Matrix<CompTy, M, N, MatrixUse::B, Scope> MatrixB);

  template <ComponentEnum LHSTy, ComponentEnum RHSTy, SIZE_TYPE K,
            MatrixUseEnum UseLocal = Use>
  typename hlsl::enable_if<Use == MatrixUse::Accumulator && UseLocal == Use,
                           void>::type
  MultiplyAccumulate(const Matrix<LHSTy, M, K, MatrixUse::A, Scope> MatrixA,
                     const Matrix<RHSTy, K, N, MatrixUse::B, Scope> MatrixB);
};

// Thread-scope matrices expose only the operations declared by this partial
// specialization.
template <ComponentEnum ComponentTy, SIZE_TYPE M, SIZE_TYPE N,
          MatrixUseEnum Use>
class Matrix<ComponentTy, M, N, Use, MatrixScope::Thread> {
  using ElementType = typename __detail::ComponentTypeTraits<ComponentTy>::Type;

  template <MatrixLayoutEnum Layout, uint Align = 128,
            MatrixUseEnum UseLocal = Use>
  static typename hlsl::enable_if<Use == MatrixUse::A && UseLocal == Use,
                                  Matrix>::type
  Load(ByteAddressBuffer Res, uint StartOffset, uint Stride);

  template <uint Align = 128, MatrixUseEnum UseLocal = Use>
  typename hlsl::enable_if<Use == MatrixUse::Accumulator && UseLocal == Use,
                           void>::type
  InterlockedAccumulate(RWByteAddressBuffer Res, uint StartOffset);
};

MatrixUseEnum AccumulatorLayout();

template <ComponentEnum OutTy, ComponentEnum ATy, ComponentEnum BTy,
          SIZE_TYPE M, SIZE_TYPE N, SIZE_TYPE K>
Matrix<OutTy, M, N, MatrixUse::Accumulator, MatrixScope::Wave>
Multiply(const Matrix<ATy, M, K, MatrixUse::A, MatrixScope::Wave> MatrixA,
         const Matrix<BTy, K, N, MatrixUse::B, MatrixScope::Wave> MatrixB);

template <ComponentEnum CompTy, SIZE_TYPE M, SIZE_TYPE N, SIZE_TYPE K>
Matrix<CompTy, M, N, MatrixUse::Accumulator, MatrixScope::Wave>
Multiply(const Matrix<CompTy, M, K, MatrixUse::A, MatrixScope::Wave> MatrixA,
         const Matrix<CompTy, K, N, MatrixUse::B, MatrixScope::Wave> MatrixB);

template <ComponentEnum OutTy, ComponentEnum ATy, ComponentEnum BTy,
          SIZE_TYPE M, SIZE_TYPE N, SIZE_TYPE K>
Matrix<OutTy, M, N, MatrixUse::Accumulator, MatrixScope::ThreadGroup> Multiply(
    const Matrix<ATy, M, K, MatrixUse::A, MatrixScope::ThreadGroup> MatrixA,
    const Matrix<BTy, K, N, MatrixUse::B, MatrixScope::ThreadGroup> MatrixB);

template <ComponentEnum CompTy, SIZE_TYPE M, SIZE_TYPE N, SIZE_TYPE K>
Matrix<CompTy, M, N, MatrixUse::Accumulator, MatrixScope::ThreadGroup> Multiply(
    const Matrix<CompTy, M, K, MatrixUse::A, MatrixScope::ThreadGroup> MatrixA,
    const Matrix<CompTy, K, N, MatrixUse::B, MatrixScope::ThreadGroup> MatrixB);

// Cooperative Vector Replacement API
// Cooperative Vector operates on per-thread vectors multiplying against A
// matrices with thread scope.

template <typename OutputElTy, typename InputElTy, SIZE_TYPE M, SIZE_TYPE K,
          ComponentEnum MatrixDT>
typename hlsl::enable_if<hlsl::is_arithmetic<InputElTy>::value,
                         vector<OutputElTy, M> >::type
Multiply(Matrix<MatrixDT, M, K, MatrixUse::A, MatrixScope::Thread> MatrixA,
         vector<InputElTy, K> Vec);

template <typename OutputElTy, typename InputElTy, ComponentEnum InputInterp,
          SIZE_TYPE M, SIZE_TYPE K, SIZE_TYPE VecK, ComponentEnum MatrixDT>
typename hlsl::enable_if<
    InterpretedVector<InputElTy, VecK, InputInterp>::Size == K,
    vector<OutputElTy, M> >::type
Multiply(Matrix<MatrixDT, M, K, MatrixUse::A, MatrixScope::Thread> MatrixA,
         InterpretedVector<InputElTy, VecK, InputInterp> InterpVec);

template <typename OutputElTy, typename InputElTy, typename BiasElTy,
          SIZE_TYPE M, SIZE_TYPE K, ComponentEnum MatrixDT>
typename hlsl::enable_if<hlsl::is_arithmetic<InputElTy>::value &&
                             hlsl::is_arithmetic<BiasElTy>::value,
                         vector<OutputElTy, M> >::type
MultiplyAdd(Matrix<MatrixDT, M, K, MatrixUse::A, MatrixScope::Thread> MatrixA,
            vector<InputElTy, K> Vec, vector<BiasElTy, M> Bias);

template <typename OutputElTy, typename InputElTy, ComponentEnum InputInterp,
          typename BiasElTy, SIZE_TYPE M, SIZE_TYPE K, SIZE_TYPE VecK,
          ComponentEnum MatrixDT>
typename hlsl::enable_if<
    VecK == __detail::ScalarCountFromPackedComponents<InputInterp, K>::Value &&
        hlsl::is_arithmetic<BiasElTy>::value,
    vector<OutputElTy, M> >::type
MultiplyAdd(Matrix<MatrixDT, M, K, MatrixUse::A, MatrixScope::Thread> MatrixA,
            InterpretedVector<InputElTy, VecK, InputInterp> InterpVec,
            vector<BiasElTy, M> Bias);

template <typename OutputElTy, typename InputElTy, ComponentEnum BiasElTy,
          SIZE_TYPE M, SIZE_TYPE K, ComponentEnum MatrixDT>
typename hlsl::enable_if<hlsl::is_arithmetic<InputElTy>::value,
                         vector<OutputElTy, M> >::type
MultiplyAdd(Matrix<MatrixDT, M, K, MatrixUse::A, MatrixScope::Thread> MatrixA,
            vector<InputElTy, K> Vec, VectorRef<BiasElTy, M> BiasRef);

template <typename OutputElTy, typename InputElTy, ComponentEnum InputInterp,
          ComponentEnum BiasElTy, SIZE_TYPE M, SIZE_TYPE K, SIZE_TYPE VecK,
          ComponentEnum MatrixDT>
typename hlsl::enable_if<
    InterpretedVector<InputElTy, VecK, InputInterp>::Size == K,
    vector<OutputElTy, M> >::type
MultiplyAdd(Matrix<MatrixDT, M, K, MatrixUse::A, MatrixScope::Thread> MatrixA,
            InterpretedVector<InputElTy, VecK, InputInterp> InterpVec,
            VectorRef<BiasElTy, M> BiasRef);

template <ComponentEnum OutTy, typename InputElTy, SIZE_TYPE M, SIZE_TYPE N>
typename hlsl::enable_if<
    hlsl::is_arithmetic<InputElTy>::value,
    Matrix<OutTy, M, N, MatrixUse::Accumulator, MatrixScope::Thread> >::type
OuterProduct(vector<InputElTy, M> VecA, vector<InputElTy, N> VecB);

template <uint Align = 64, typename InputElTy, SIZE_TYPE M>
typename hlsl::enable_if<hlsl::is_arithmetic<InputElTy>::value, void>::type
InterlockedAccumulate(RWByteAddressBuffer Res,
                      uint StartOffset, vector<InputElTy, M> Vec);

} // namespace linalg
} // namespace dx
```

### Example Usage: Wave Matrix

```c++
RWByteAddressBuffer B : register(u0);

groupshared float GSMat[128];
groupshared vector<float, 8> GSMatVec[16];
#ifdef __hlsl_dx_compiler
groupshared uint8_t4_packed GSMatPacked[32];
groupshared vector<uint8_t4_packed, 4> GSMatPackedVec[16];
#endif

void WaveMatrixExample() {
  using namespace dx::linalg;
  using MatrixATy =
      Matrix<ComponentType::F16, 8, 32, MatrixUse::A, MatrixScope::Wave>;
  using MatrixBTy =
      Matrix<ComponentType::F16, 32, 16, MatrixUse::B, MatrixScope::Wave>;
  using MatrixAccumTy = Matrix<ComponentType::F16, 8, 16,
                               MatrixUse::Accumulator, MatrixScope::Wave>;
  using MatrixAccum32Ty = Matrix<ComponentType::F32, 8, 16,
                                 MatrixUse::Accumulator, MatrixScope::Wave>;

  MatrixATy MatA = MatrixATy::Load(
      B, 0, /* Row stride = number of columns * element size */ 32 * 2,
      MatrixLayout::RowMajor);
  MatrixBTy MatB = MatrixBTy::Load(
      B, 0, /* Row stride = number of columns * element size */ 16 * 2,
      MatrixLayout::RowMajor);

  for (uint I = 0; I < MatB.Length(); ++I) {
    uint2 Pos = MatB.GetCoordinate(I);
    // Apply `tanh` to each off-diagonal component.
    if (Pos.x != Pos.y) {
      float16_t Val = MatB.Get(I);
      MatB.Set(I, tanh(Val));
    }
  }

  MatrixAccumTy Accum = Multiply(MatA, MatB);
  MatrixAccum32Ty Accum32 = Multiply<ComponentType::F32>(MatA, MatB);
  MatrixAccum32Ty M =
      MatrixAccum32Ty::Load(GSMat, 0, 8, MatrixLayout::RowMajor);
  M.Store(GSMat, 0, 8, MatrixLayout::RowMajor);

  MatrixAccum32Ty M2 =
      MatrixAccum32Ty::Load(GSMatVec, 0, 8, MatrixLayout::RowMajor);
  M2.MultiplyAccumulate(MatA, MatB);
  M2.Store(GSMatVec, 0, 8, MatrixLayout::RowMajor);
#ifdef __hlsl_dx_compiler
  MatrixAccumTy M16 =
      MatrixAccumTy::Load(GSMatPackedVec, 0, 8, MatrixLayout::RowMajor);

  // The next line is an error because the groupshared array isn't the right
  // type for the matrix component.
  //
  // MatrixAccumTy M2 = MatrixAccumTy::Load(GSMat, 0, 8,
  // MatrixLayout::RowMajor);

  M.Store(GSMat, 0, 8, MatrixLayout::RowMajor);
  M.InterlockedAccumulate(GSMat, 0, 8, MatrixLayout::RowMajor);

  M.InterlockedAccumulate<ComponentType::I8>(GSMatPacked, 0, 2,
                                             MatrixLayout::RowMajor);
  M.InterlockedAccumulate<ComponentType::I8>(GSMatPackedVec, 0, 2,
                                             MatrixLayout::RowMajor);
#endif
}
```

### Example Usage: Cooperative Vectors

```c++
ByteAddressBuffer MBuf : register(t0);
RWByteAddressBuffer VAccumBuf : register(u0);

void CoopVec() {
  using namespace dx::linalg;
  using MatrixATy =
      Matrix<ComponentType::F16, 16, 16, MatrixUse::A, MatrixScope::Thread>;

  vector<float16_t, 16> Vec = (vector<float16_t, 16>)0;
  MatrixATy MatA = MatrixATy::Load<MatrixLayout::RowMajor>(
      MBuf, 0, /* Row stride = number of columns * element size */ 16 * 2);
  vector<float16_t, 16> Layer1 = Multiply<float16_t>(MatA, Vec);

  vector<float16_t, 16> NullBias = (vector<float16_t, 16>)0;
  vector<float16_t, 16> Layer2 = MultiplyAdd<float16_t>(MatA, Layer1, NullBias);

  VectorRef<ComponentType::F8_E4M3FN, 16> MemBias = {MBuf,
                                                     /*start offset*/ 4096};
  vector<float16_t, 16> Layer3 = MultiplyAdd<float16_t>(MatA, Layer2, MemBias);

  // Clang doesn't yet support packed types.
#ifdef __hlsl_dx_compiler
  vector<uint8_t4_packed, 4> SomeData = (vector<uint8_t4_packed, 4>)0;

  vector<float16_t, 16> Layer4 = MultiplyAdd<float16_t>(
      MatA, MakeInterpretedVector<ComponentType::F8_E4M3FN>(SomeData), MemBias);
  vector<float16_t, 16> Layer5 = MultiplyAdd<float16_t>(
      MatA, MakeInterpretedVector<ComponentType::F8_E4M3FN>(SomeData),
      NullBias);

  vector<float16_t, 16> Layer6 = MultiplyAdd<float16_t>(
      MatA, MakeInterpretedVector<ComponentType::F8_E4M3FN>(SomeData), MemBias);

  // This example creates an interpreted vector where the data needs to be
  // converted from a source type to a destination type.
  vector<uint, 16> SomeData2 = (vector<uint, 16>)0;
  vector<float16_t, 16> Layer7 = MultiplyAdd<float16_t>(
      MatA, Convert<ComponentType::F8_E4M3FN, ComponentType::U32>(SomeData2),
      MemBias);
#endif
  InterlockedAccumulate</*Align=*/128>(VAccumBuf, 128, Layer3);
}
```

### Example Usage: OuterProduct and InterlockedAccumulate

```c++
RWByteAddressBuffer Buf : register(u1);

void OuterProdAccum() {
  using namespace dx::linalg;
  using MatrixAccumTy = Matrix<ComponentType::F16, 16, 8,
                               MatrixUse::Accumulator, MatrixScope::Thread>;

  vector<float16_t, 16> VecA = (vector<float16_t, 16>)0;
  vector<float16_t, 8> VecB = (vector<float16_t, 8>)0;
  MatrixAccumTy MatAcc = OuterProduct<ComponentType::F16>(VecA, VecB);

  MatAcc.InterlockedAccumulate</*Align=*/128>(Buf, 0);
}
```

## Detailed design

### D3D API Additions

#### Resource Alignment Requirements

Buffers passed to linear-algebra intrinsics must meet the following base-address
and size guarantees. The HLSL requirements in [HLSL Additions](#hlsl-additions)
describe the minimum alignment that an intrinsic can encode. They do not relax
the D3D runtime requirements below. An application must satisfy both sets of
requirements; where they differ, the stricter requirement applies.

##### Matrix buffers (Multiply / MultiplyAdd / OuterProduct)

For any buffer serving as a matrix source or destination in a linear-algebra multiply, multiply-add, outer product, or matrix accumulate operation:

* The buffer's base GPU virtual address, and any offset within it used as the matrix start, must be aligned to at least **128 bytes**.
* The matrix stride must be aligned to at least **16 bytes**.
* The buffer size must be a multiple of **16 bytes**, so that the 16-byte access covering the last row or column of the matrix is guaranteed to touch valid memory.

The 128-byte and 16-byte requirements on `DestVA`, `DestSize`, and `DestStride`
for [`ConvertLinearAlgebraMatrix`](#convert-matrix-layout) apply the same rule to
the conversion destination.

##### Atomic accumulate store buffers

For any buffer used as the destination of a vector atomic-accumulate-store operation:

* The buffer's base GPU virtual address, and any offset within it used as the array start, must be aligned to at least **64 bytes**.
* The buffer size must be a multiple of **16 bytes**. Implementations may write into the padding between the end of the array and the next 16-byte boundary, so applications must not use that padding space for any other purpose.

#### Granular Capability Query API

While the tier system provides a convenient way to target well-defined hardware profiles, some applications may need to query support for specific matrix operation configurations that fall outside standard tier definitions or to leverage vendor-specific capabilities. The D3D12 runtime exposes two complementary capability query APIs:

* The **granular capability query API** described in this section answers the targeted question *"is this specific configuration supported?"* It is the natural fit for runtime validation (for example, checking whether a precompiled shader's matrix operation can be executed on the current device).
* The [Operation Enumeration API](#operation-enumeration-api) answers the
  discovery question *"which configurations does the driver support, and which
  tile shapes are native?"* It is the natural fit for applications that want
  to select an operation shape or data-type combination from the driver's
  capabilities rather than test a specific one.

Both APIs report the same underlying capabilities; an application is free to use either or both.

##### D3D12_LINEAR_ALGEBRA_OPERATION_TYPE

```cpp
typedef enum D3D12_LINEAR_ALGEBRA_OPERATION_TYPE
{
    D3D12_LINEAR_ALGEBRA_OPERATION_TYPE_MATRIX_CONSTRUCTION,
    D3D12_LINEAR_ALGEBRA_OPERATION_TYPE_WAVE_MATRIX_MULTIPLY,
    D3D12_LINEAR_ALGEBRA_OPERATION_TYPE_THREADGROUP_MATRIX_MULTIPLY,
    D3D12_LINEAR_ALGEBRA_OPERATION_TYPE_THREAD_VECTOR_MATRIX_MULTIPLY,
    D3D12_LINEAR_ALGEBRA_OPERATION_TYPE_THREAD_OUTER_PRODUCT,
    D3D12_LINEAR_ALGEBRA_OPERATION_TYPE_ATOMIC_ACCUMULATE_STORE,
} D3D12_LINEAR_ALGEBRA_OPERATION_TYPE;
```

##### D3D12_LINEAR_ALGEBRA_DATATYPE
```cpp
typedef enum D3D12_LINEAR_ALGEBRA_DATATYPE {
    D3D12_LINEAR_ALGEBRA_DATATYPE_NONE = 0,
    D3D12_LINEAR_ALGEBRA_DATATYPE_SINT16 = 2,
    D3D12_LINEAR_ALGEBRA_DATATYPE_UINT16 = 3,
    D3D12_LINEAR_ALGEBRA_DATATYPE_SINT32 = 4,
    D3D12_LINEAR_ALGEBRA_DATATYPE_UINT32 = 5,
    D3D12_LINEAR_ALGEBRA_DATATYPE_FLOAT16 = 7,
    D3D12_LINEAR_ALGEBRA_DATATYPE_FLOAT32 = 8,
    D3D12_LINEAR_ALGEBRA_DATATYPE_SINT8 = 18,
    D3D12_LINEAR_ALGEBRA_DATATYPE_UINT8 = 19,
    D3D12_LINEAR_ALGEBRA_DATATYPE_FLOAT8_E4M3FN = 20,
    D3D12_LINEAR_ALGEBRA_DATATYPE_FLOAT8_E5M2 = 21
} D3D12_LINEAR_ALGEBRA_DATATYPE;
```

##### D3D12_LINEAR_ALGEBRA_OPERATION_SUPPORT_QUERY

###### D3D12_LINEAR_ALGEBRA_MATRIX_SHAPE
``` cpp
typedef struct D3D12_LINEAR_ALGEBRA_MATRIX_SHAPE
{
    UINT M;  // Rows in matrix A
    UINT K;  // Columns in matrix A / Rows in matrix B
    UINT N;  // Columns in matrix B
} D3D12_LINEAR_ALGEBRA_MATRIX_SHAPE;
```

- `M`, `K`, `N` - Matrix dimensions following the formula MxK * KxN = MxN. Each matrix shape simultaneously names a supported A matrix (MxK), B matrix (KxN), and accumulator matrix (MxN) layout. The same shape type is used for both matrix construction and matrix multiplication queries.

###### D3D12_LINEAR_ALGEBRA_MATRIX_CONSTRUCTION_SUPPORT
``` cpp
typedef struct D3D12_LINEAR_ALGEBRA_MATRIX_CONSTRUCTION_SUPPORT
{
    // Inputs
    D3D12_LINEAR_ALGEBRA_DATATYPE ComponentType;
    UINT WaveSize;
    D3D12_LINEAR_ALGEBRA_MATRIX_SHAPE Shape;

    // Outputs
    BOOL Supported;
} D3D12_LINEAR_ALGEBRA_MATRIX_CONSTRUCTION_SUPPORT;
```

This query reports support for general operations on wave-scope and
threadgroup-scope matrices. Multiplication support alone is insufficient because
matrices at these scopes can be loaded, stored, manipulated, and converted
without participating in multiplication. A positive result indicates that the
driver can represent the requested component type and shape. It must support:
* Loading a matrix of that type and shape from buffer or group-shared memory, and similarly for storing (`Load()`/`Store()`).
* Operating on elements of a matrix (`Length()`/`GetCoordinate()`/`Get()`/`Set()`/`Splat()`).
* Being used as a source or destination of a conversion (`Cast()`).

- `ComponentType` - The matrix component type being queried.

- `WaveSize` - The wave size for the shader constructing the matrix. Must be a power of 2 in the device's valid wave size range, or else 0 to indicate any.

- `Shape` - Simultaneously identifies the A matrix ($M \times K$), B matrix
  ($K \times N$), and accumulator matrix ($M \times N$). `Supported` is `TRUE`
  only if the driver can construct all three matrices with `ComponentType`.
  Application shapes that are an integer multiple of a native shape in each
  dimension are supported. Applications can query a native shape discovered
  through the [Operation Enumeration API](#operation-enumeration-api), or query
  the shape they intend to construct directly.

- `Supported` - On output, `TRUE` if the driver can construct the requested shape for the requested component type.

For any `(ComponentType, Shape)` reported as supported by the [wave-scope](#d3d12_linear_algebra_wave_matrix_multiply_support) or [threadgroup-scope](#d3d12_linear_algebra_threadgroup_matrix_multiply_support) multiplication queries, this query must also report support for that component type and shape -- every matrix that can participate in a supported multiplication must also be constructible. Drivers that support multiple multiplication tilings for the same type combination (for example 4x16x16 alongside 16x4x16) implicitly support construction at each tiling.

###### D3D12_LINEAR_ALGEBRA_WAVE_MATRIX_MULTIPLY_INPUTS
``` cpp
typedef struct D3D12_LINEAR_ALGEBRA_WAVE_MATRIX_MULTIPLY_INPUTS
{
    UINT WaveSize;
    D3D12_LINEAR_ALGEBRA_DATATYPE MatrixAComponentType;
    D3D12_LINEAR_ALGEBRA_DATATYPE MatrixBComponentType;
    D3D12_LINEAR_ALGEBRA_DATATYPE AccumulatorComponentType;
} D3D12_LINEAR_ALGEBRA_WAVE_MATRIX_MULTIPLY_INPUTS;
```

- `WaveSize` - The wave size for the shader executing the operation. Must be a power of 2 in the device's valid wave size range, or else 0 to indicate any.

- `MatrixAComponentType` - Component type of the A matrix.

- `MatrixBComponentType` - Component type of the B matrix.

- `AccumulatorComponentType` - Component type of the accumulator matrix.

###### D3D12_LINEAR_ALGEBRA_MULTIPLICATION_SUPPORT_FLAGS

```cpp
typedef enum D3D12_LINEAR_ALGEBRA_MULTIPLICATION_SUPPORT_FLAGS
{
    D3D12_LINEAR_ALGEBRA_MULTIPLICATION_SUPPORT_FLAG_SUPPORTED = 1,
    D3D12_LINEAR_ALGEBRA_MULTIPLICATION_SUPPORT_FLAG_EMULATED_INPUTS = 2,
    D3D12_LINEAR_ALGEBRA_MULTIPLICATION_SUPPORT_FLAG_EMULATED_OUTPUTS = 4,
    D3D12_LINEAR_ALGEBRA_MULTIPLICATION_SUPPORT_FLAG_TRANSPOSE = 8,
} D3D12_LINEAR_ALGEBRA_MULTIPLICATION_SUPPORT_FLAGS;
```

|Flag|Meaning|
|----|-------|
|`D3D12_LINEAR_ALGEBRA_MULTIPLICATION_SUPPORT_FLAG_SUPPORTED`|The driver will accept the operation in a shader.|
|`D3D12_LINEAR_ALGEBRA_MULTIPLICATION_SUPPORT_FLAG_EMULATED_INPUTS`|The hardware does not accelerate multiplication of the specified input types. This flag applies only to thread-scope vector-matrix multiplication and may be present only when `MatrixInputType` is `FLOAT8_E4M3FN` or `FLOAT8_E5M2`. When this flag is present, the operation may use higher internal precision; in particular, the implementation need not convert the vector input to the matrix type before multiplication. The input matrix data MUST use `D3D12_LINEAR_ALGEBRA_MATRIX_LAYOUT_MUL_OPTIMAL`, or `D3D12_LINEAR_ALGEBRA_MATRIX_LAYOUT_MUL_OPTIMAL_TRANSPOSE` when `D3D12_LINEAR_ALGEBRA_MULTIPLICATION_SUPPORT_FLAG_TRANSPOSE` is also present.|
|`D3D12_LINEAR_ALGEBRA_MULTIPLICATION_SUPPORT_FLAG_EMULATED_OUTPUTS`|The hardware does not have acceleration for accumulation of the specified output types. If this flag is present, then the operation may be performed with higher or lower internal precision, followed by a final conversion step. For example, if present on an operation with `FLOAT16` inputs and outputs, then the accumulation step for each individual `FLOAT16` output may be performed at `FLOAT32` precision with a final rounding step, rather than `FLOAT16` rounding after each multiply-add step in the dot product of the outputs.|
|`D3D12_LINEAR_ALGEBRA_MULTIPLICATION_SUPPORT_FLAG_TRANSPOSE`|The driver can perform loads through transposed layouts. This is only relevant for thread-scope vector-matrix multiplication.|

###### D3D12_LINEAR_ALGEBRA_WAVE_MATRIX_MULTIPLY_SUPPORT
``` cpp
typedef struct D3D12_LINEAR_ALGEBRA_WAVE_MATRIX_MULTIPLY_SUPPORT
{
    // Inputs
    D3D12_LINEAR_ALGEBRA_WAVE_MATRIX_MULTIPLY_INPUTS Inputs;
    D3D12_LINEAR_ALGEBRA_MATRIX_SHAPE Shape;

    // Outputs
    D3D12_LINEAR_ALGEBRA_MULTIPLICATION_SUPPORT_FLAGS SupportFlags;
} D3D12_LINEAR_ALGEBRA_WAVE_MATRIX_MULTIPLY_SUPPORT;
```

- `Inputs` - The type combination being queried.

- `Shape` - The matrix shape being queried. Application matrix shapes that are an integer multiple of any native shape in each dimension are reported as supported. There is no limitation on maximum matrix size, though larger matrices may result in significantly adverse performance. Native shapes can be discovered via the [Operation Enumeration API](#operation-enumeration-api).

- `SupportFlags` - Indicates whether the operation is supported, and whether any emulation of the requested data types would occur. See [D3D12_LINEAR_ALGEBRA_MULTIPLICATION_SUPPORT_FLAGS](#d3d12_linear_algebra_multiplication_support_flags) for the meaning of each flag.

###### D3D12_LINEAR_ALGEBRA_THREADGROUP_MATRIX_MULTIPLY_SUPPORT
``` cpp
typedef struct D3D12_LINEAR_ALGEBRA_THREADGROUP_MATRIX_MULTIPLY_SUPPORT
{
    // Inputs
    D3D12_LINEAR_ALGEBRA_WAVE_MATRIX_MULTIPLY_INPUTS WaveInputs;
    D3D12_LINEAR_ALGEBRA_MATRIX_SHAPE Shape;

    // Outputs
    D3D12_LINEAR_ALGEBRA_MULTIPLICATION_SUPPORT_FLAGS SupportFlags;
    UINT MinThreadGroupSize;
    UINT MaxThreadGroupSize;
    UINT PreferredThreadGroupSize;
} D3D12_LINEAR_ALGEBRA_THREADGROUP_MATRIX_MULTIPLY_SUPPORT;
```

- `WaveInputs` - The type combination being queried.

- `Shape` - The matrix shape being queried. Application matrix shapes that are an integer multiple of any native shape in each dimension are reported as supported. There is no limitation on maximum matrix size, though larger matrices may result in significantly adverse performance. Native shapes can be discovered via the [Operation Enumeration API](#operation-enumeration-api).

- `SupportFlags` - Indicates whether the operation is supported, and whether any emulation of the requested data types would occur. See [D3D12_LINEAR_ALGEBRA_MULTIPLICATION_SUPPORT_FLAGS](#d3d12_linear_algebra_multiplication_support_flags) for the meaning of each flag.

- `MinThreadGroupSize` - The minimum number of threads in a group that can perform this multiplication for the requested shape.

- `MaxThreadGroupSize` - The maximum number of threads in a group that can perform this multiplication for the requested shape. Valid sizes are then multiples of the minimum, up to and including the maximum.

- `PreferredThreadGroupSize` - The driver's estimate for the most efficient thread group size to perform this multiplication for the requested shape. This may be zero, indicating that there are trade-offs (e.g. register pressure vs throughput) and it is not possible for the driver to report an optimal size.

###### D3D12_LINEAR_ALGEBRA_THREAD_VECTOR_MATRIX_MULTIPLY_SUPPORT
``` cpp
typedef struct D3D12_LINEAR_ALGEBRA_THREAD_VECTOR_MATRIX_MULTIPLY_SUPPORT
{
    // Inputs
    D3D12_LINEAR_ALGEBRA_DATATYPE VectorInputType;
    D3D12_LINEAR_ALGEBRA_DATATYPE MatrixInputType;
    D3D12_LINEAR_ALGEBRA_DATATYPE BiasInputType;
    D3D12_LINEAR_ALGEBRA_DATATYPE VectorResultType;

    // Outputs
    D3D12_LINEAR_ALGEBRA_MULTIPLICATION_SUPPORT_FLAGS SupportFlags;
} D3D12_LINEAR_ALGEBRA_THREAD_VECTOR_MATRIX_MULTIPLY_SUPPORT;
```

- `VectorInputType` - The *interpretation type* of the input vector operand as consumed by the DXIL multiply. The application produces a vector at this interpretation via one of three paths (see [Supplying the vector operand](#supplying-the-vector-operand) below).

- `MatrixInputType` - The component type of the input matrix.

- `BiasInputType` - The type of data that's added to the multiplication result before returning the result.

- `VectorResultType` - The type of the bias and result vectors.

- `SupportFlags` - Indicates level of support for this operation. See [D3D12_LINEAR_ALGEBRA_MULTIPLICATION_SUPPORT_FLAGS](#d3d12_linear_algebra_multiplication_support_flags) for the meaning of each flag.

###### Supplying the vector operand

`VectorInputType` and `MatrixInputType` are independent parameters of the caps query: the HLSL and DXIL specifications place no matching constraint between them. The application produces an input vector at the interpretation `VectorInputType` via one of three paths:

* **Native HLSL vector.** The vector's HLSL element type *is* the interpretation. `VectorInputType` equals the vector's element type.
* **`linalg::MakeInterpretedVector<DT>(vec)`.** A re-typing wrapper around a native vector that already contains data laid out for the target interpretation (for example, an FP8 payload stored packed in `uint32_t`). `MakeInterpretedVector` does not convert data; `VectorInputType` equals `DT`.
* **`linalg::Convert<DestTy>(vec)`.** Actually performs a conversion per the HLSL linear-algebra conversion rules and produces an `InterpretedVector<..., DestTy>`. `VectorInputType` equals `DestTy`.

The `EMULATED_INPUTS` flag is the mechanism by which the driver signals that a supported combination is not run natively at `MatrixInputType` precision — for example, when the input interpretation differs from the matrix type beyond integer signedness, or when the matrix type is FP8 and the driver must widen internally. When set, the multiply may execute at higher precision than `MatrixInputType`. Similarly `EMULATED_OUTPUTS` signals internal precision higher than `VectorResultType` prior to down-conversion. Neither flag set means the operation is natively accelerated end-to-end.

###### `linalg::Convert` support

There is no separate capability query for `linalg::Convert`. Any driver implementing linear algebra supports every conversion whose source type it otherwise natively supports in HLSL and whose destination type appears as a supported `VectorInputType` in some vector-matrix multiply configuration, following the conversion rules documented in [the HLSL Data Conversion Rules](#data-conversion-rules).

###### D3D12_LINEAR_ALGEBRA_THREAD_OUTER_PRODUCT_SUPPORT

``` cpp
typedef struct D3D12_LINEAR_ALGEBRA_THREAD_OUTER_PRODUCT_SUPPORT
{
    // Inputs
    D3D12_LINEAR_ALGEBRA_DATATYPE InputComponentType;
    D3D12_LINEAR_ALGEBRA_DATATYPE ResultComponentType;

    // Outputs
    BOOL Supported;
} D3D12_LINEAR_ALGEBRA_THREAD_OUTER_PRODUCT_SUPPORT;
```

- `InputComponentType` - Type of the input vectors. Both vectors must have the same type.
- `ResultComponentType` - Type of the output vector.
- `Supported` - Output: Whether the outer product operation is supported.

###### D3D12_LINEAR_ALGEBRA_ATOMIC_ACCUMULATE_STORE_SUPPORT

``` cpp
typedef struct D3D12_LINEAR_ALGEBRA_ATOMIC_ACCUMULATE_STORE_SUPPORT
{
    // Inputs
    D3D12_LINEAR_ALGEBRA_DATATYPE ComponentType;

    // Outputs
    BOOL RWByteAddressBufferSupported;
    BOOL GroupSharedSupported;
} D3D12_LINEAR_ALGEBRA_ATOMIC_ACCUMULATE_STORE_SUPPORT;
```

- `ComponentType` - Type of the input matrix or vector.
- `RWByteAddressBufferSupported` - Whether this atomic operation is supported on UAVs.
- `GroupSharedSupported` - Whether this atomic operation is supported on group-shared arrays.

###### D3D12_FEATURE_DATA_LINEAR_ALGEBRA_MATRIX_OPERATION_SUPPORT

```cpp
typedef struct D3D12_FEATURE_DATA_LINEAR_ALGEBRA_MATRIX_OPERATION_SUPPORT
{
    D3D12_LINEAR_ALGEBRA_OPERATION_TYPE OperationType;
    union
    {
        D3D12_LINEAR_ALGEBRA_MATRIX_CONSTRUCTION_SUPPORT MatrixConstruction;
        D3D12_LINEAR_ALGEBRA_WAVE_MATRIX_MULTIPLY_SUPPORT WaveMatrixMultiply;
        D3D12_LINEAR_ALGEBRA_THREADGROUP_MATRIX_MULTIPLY_SUPPORT ThreadGroupMatrixMultiply;
        D3D12_LINEAR_ALGEBRA_THREAD_VECTOR_MATRIX_MULTIPLY_SUPPORT ThreadVectorMatrixMultiply;
        D3D12_LINEAR_ALGEBRA_THREAD_OUTER_PRODUCT_SUPPORT ThreadOuterProductSupport;
        D3D12_LINEAR_ALGEBRA_ATOMIC_ACCUMULATE_STORE_SUPPORT AccumulateStore;
    };
} D3D12_FEATURE_DATA_LINEAR_ALGEBRA_MATRIX_OPERATION_SUPPORT;
```
Members:

- `OperationType` - The type of operation to query support for.

- `MatrixConstruction` - Used when `OperationType` is `D3D12_LINEAR_ALGEBRA_OPERATION_TYPE_MATRIX_CONSTRUCTION`.

- `WaveMatrixMultiply` - Used when `OperationType` is `D3D12_LINEAR_ALGEBRA_OPERATION_TYPE_WAVE_MATRIX_MULTIPLY`.

- `ThreadGroupMatrixMultiply` - Used when `OperationType` is `D3D12_LINEAR_ALGEBRA_OPERATION_TYPE_THREADGROUP_MATRIX_MULTIPLY`.

- `ThreadVectorMatrixMultiply` - Used when `OperationType` is `D3D12_LINEAR_ALGEBRA_OPERATION_TYPE_THREAD_VECTOR_MATRIX_MULTIPLY`.

- `ThreadOuterProductSupport` - Used when `OperationType` is `D3D12_LINEAR_ALGEBRA_OPERATION_TYPE_THREAD_OUTER_PRODUCT`. Output formats from outer product must be supported for accumulate-store.

- `AccumulateStore` - Used when `OperationType` is `D3D12_LINEAR_ALGEBRA_OPERATION_TYPE_ATOMIC_ACCUMULATE_STORE`.

##### Usage Example
``` cpp
// I have a shader that tries to use wave scope matrix multiplication with MxKxN = 64x64x64, FP16xFP16->FP32.
// Query if it's valid for me to use that shader.
D3D12_FEATURE_DATA_LINEAR_ALGEBRA_MATRIX_OPERATION_SUPPORT opSupport = {};
opSupport.OperationType = D3D12_LINEAR_ALGEBRA_OPERATION_TYPE_WAVE_MATRIX_MULTIPLY;
opSupport.WaveMatrixMultiply.Inputs.WaveSize = 0; // I don't care about the wave size
opSupport.WaveMatrixMultiply.Inputs.MatrixAComponentType = D3D12_LINEAR_ALGEBRA_DATATYPE_FLOAT16;
opSupport.WaveMatrixMultiply.Inputs.MatrixBComponentType = D3D12_LINEAR_ALGEBRA_DATATYPE_FLOAT16;
opSupport.WaveMatrixMultiply.Inputs.AccumulatorComponentType = D3D12_LINEAR_ALGEBRA_DATATYPE_FLOAT32;
opSupport.WaveMatrixMultiply.Shape = { 64, 64, 64 };

HRESULT hr = device->CheckFeatureSupport(
    D3D12_FEATURE_LINEAR_ALGEBRA_MATRIX_OPERATION_SUPPORT,
    &opSupport,
    sizeof(opSupport));

if (SUCCEEDED(hr) &&
    (opSupport.WaveMatrixMultiply.SupportFlags & D3D12_LINEAR_ALGEBRA_MULTIPLICATION_SUPPORT_FLAG_SUPPORTED))
{
    // 64x64x64 FP16xFP16->FP32 is supported on this device. The driver will internally
    // tile the operation using one of its native shapes; see the Operation Enumeration
    // API if the application needs to know which native shapes are available.
}

// Query vector-matrix multiply support
D3D12_FEATURE_DATA_LINEAR_ALGEBRA_MATRIX_OPERATION_SUPPORT vecMatSupport = {};
vecMatSupport.OperationType = D3D12_LINEAR_ALGEBRA_OPERATION_TYPE_THREAD_VECTOR_MATRIX_MULTIPLY;
vecMatSupport.ThreadVectorMatrixMultiply.VectorInputType = D3D12_LINEAR_ALGEBRA_DATATYPE_FLOAT16;
vecMatSupport.ThreadVectorMatrixMultiply.MatrixInputType = D3D12_LINEAR_ALGEBRA_DATATYPE_FLOAT16;
vecMatSupport.ThreadVectorMatrixMultiply.BiasInputType = D3D12_LINEAR_ALGEBRA_DATATYPE_NONE;
vecMatSupport.ThreadVectorMatrixMultiply.VectorResultType = D3D12_LINEAR_ALGEBRA_DATATYPE_FLOAT16;

hr = device->CheckFeatureSupport(
    D3D12_FEATURE_LINEAR_ALGEBRA_MATRIX_OPERATION_SUPPORT,
    &vecMatSupport,
    sizeof(vecMatSupport));

if (SUCCEEDED(hr) &&
    (vecMatSupport.ThreadVectorMatrixMultiply.SupportFlags & D3D12_LINEAR_ALGEBRA_MULTIPLICATION_SUPPORT_FLAG_SUPPORTED))
{
    // Device supports this vector-matrix operation configuration
}
```

#### Operation Enumeration API

The granular query API in the previous section answers the question *"is this
exact configuration supported?"* This is useful for runtime validation but
inconvenient for applications that want to discover supported configurations
without iterating the entire cross-product of types, shapes, and wave sizes.
The Operation Enumeration API returns a flat list of supported configurations
for a given operation type.

For operations with matrix shapes, each entry contains a native tile shape.
The entry's type combination may be native or emulated, as indicated by its
support flags. Every enumerated configuration, including each integer multiple
of an enumerated shape, must be supported by the granular query. Conversely,
the granular query reports only configurations covered by the enumeration
after applying the integer-multiple rule.

##### Enumeration Entry Structures

Each enumeration entry describes one fully specified supported configuration.
For operation types whose support uses tile shapes (matrix construction,
wave-scope multiply, and threadgroup-scope multiply), the enumeration contains
one entry per `(type combination, native tile shape)` pair. Drivers that support
multiple tilings for the same type combination report each as a separate entry.
Configurations that emulate a type combination are included with the relevant
`EMULATED_INPUTS` or `EMULATED_OUTPUTS` flag in `SupportFlags`.

For operation types that depend on wave size, each entry reports the inclusive range `[MinWaveSize, MaxWaveSize]` over which the rest of the entry's fields apply. Every power-of-2 wave size in that range that also lies in the device's valid wave size range is supported by the entry. Drivers whose support is not contiguous in wave size emit a separate entry per contiguous range.

```cpp
typedef struct D3D12_LINEAR_ALGEBRA_MATRIX_CONSTRUCTION_ENUMERATION_ENTRY
{
    D3D12_LINEAR_ALGEBRA_DATATYPE ComponentType;
    UINT MinWaveSize;
    UINT MaxWaveSize;
    D3D12_LINEAR_ALGEBRA_MATRIX_SHAPE Shape;
} D3D12_LINEAR_ALGEBRA_MATRIX_CONSTRUCTION_ENUMERATION_ENTRY;

typedef struct D3D12_LINEAR_ALGEBRA_WAVE_MATRIX_MULTIPLY_ENUMERATION_ENTRY
{
    UINT MinWaveSize;
    UINT MaxWaveSize;
    D3D12_LINEAR_ALGEBRA_DATATYPE MatrixAComponentType;
    D3D12_LINEAR_ALGEBRA_DATATYPE MatrixBComponentType;
    D3D12_LINEAR_ALGEBRA_DATATYPE AccumulatorComponentType;
    D3D12_LINEAR_ALGEBRA_MULTIPLICATION_SUPPORT_FLAGS SupportFlags;
    D3D12_LINEAR_ALGEBRA_MATRIX_SHAPE Shape;
} D3D12_LINEAR_ALGEBRA_WAVE_MATRIX_MULTIPLY_ENUMERATION_ENTRY;

typedef struct D3D12_LINEAR_ALGEBRA_THREADGROUP_MATRIX_MULTIPLY_ENUMERATION_ENTRY
{
    UINT MinWaveSize;
    UINT MaxWaveSize;
    D3D12_LINEAR_ALGEBRA_DATATYPE MatrixAComponentType;
    D3D12_LINEAR_ALGEBRA_DATATYPE MatrixBComponentType;
    D3D12_LINEAR_ALGEBRA_DATATYPE AccumulatorComponentType;
    D3D12_LINEAR_ALGEBRA_MULTIPLICATION_SUPPORT_FLAGS SupportFlags;
    D3D12_LINEAR_ALGEBRA_MATRIX_SHAPE Shape;
    UINT MinThreadGroupSize;
    UINT MaxThreadGroupSize;
    UINT PreferredThreadGroupSize;
} D3D12_LINEAR_ALGEBRA_THREADGROUP_MATRIX_MULTIPLY_ENUMERATION_ENTRY;

typedef struct D3D12_LINEAR_ALGEBRA_THREAD_VECTOR_MATRIX_MULTIPLY_ENUMERATION_ENTRY
{
    D3D12_LINEAR_ALGEBRA_DATATYPE VectorInputType;
    D3D12_LINEAR_ALGEBRA_DATATYPE MatrixInputType;
    D3D12_LINEAR_ALGEBRA_DATATYPE BiasInputType;
    D3D12_LINEAR_ALGEBRA_DATATYPE VectorResultType;
    D3D12_LINEAR_ALGEBRA_MULTIPLICATION_SUPPORT_FLAGS SupportFlags;
} D3D12_LINEAR_ALGEBRA_THREAD_VECTOR_MATRIX_MULTIPLY_ENUMERATION_ENTRY;

typedef struct D3D12_LINEAR_ALGEBRA_THREAD_OUTER_PRODUCT_ENUMERATION_ENTRY
{
    D3D12_LINEAR_ALGEBRA_DATATYPE InputComponentType;
    D3D12_LINEAR_ALGEBRA_DATATYPE ResultComponentType;
} D3D12_LINEAR_ALGEBRA_THREAD_OUTER_PRODUCT_ENUMERATION_ENTRY;

typedef struct D3D12_LINEAR_ALGEBRA_ATOMIC_ACCUMULATE_STORE_ENUMERATION_ENTRY
{
    D3D12_LINEAR_ALGEBRA_DATATYPE ComponentType;
    BOOL RWByteAddressBufferSupported;
    BOOL GroupSharedSupported;
} D3D12_LINEAR_ALGEBRA_ATOMIC_ACCUMULATE_STORE_ENUMERATION_ENTRY;
```

##### D3D12_FEATURE_DATA_LINEAR_ALGEBRA_OPERATION_ENUMERATION

The new feature value `D3D12_FEATURE_LINEAR_ALGEBRA_OPERATION_ENUMERATION` is queried with a struct that selects an operation type and provides a typed pointer to the caller's entry array. `NumEntries` lives outside the union -- it carries the array capacity on input and the number of entries the driver would write on output, in entries (not bytes), regardless of which operation type is selected. The runtime writes up to `min(input capacity, available)` entries.

```cpp
typedef struct D3D12_FEATURE_DATA_LINEAR_ALGEBRA_OPERATION_ENUMERATION
{
    D3D12_LINEAR_ALGEBRA_OPERATION_TYPE OperationType;
    UINT NumEntries;
    union
    {
        D3D12_LINEAR_ALGEBRA_MATRIX_CONSTRUCTION_ENUMERATION_ENTRY          *MatrixConstruction;
        D3D12_LINEAR_ALGEBRA_WAVE_MATRIX_MULTIPLY_ENUMERATION_ENTRY         *WaveMatrixMultiply;
        D3D12_LINEAR_ALGEBRA_THREADGROUP_MATRIX_MULTIPLY_ENUMERATION_ENTRY  *ThreadGroupMatrixMultiply;
        D3D12_LINEAR_ALGEBRA_THREAD_VECTOR_MATRIX_MULTIPLY_ENUMERATION_ENTRY *ThreadVectorMatrixMultiply;
        D3D12_LINEAR_ALGEBRA_THREAD_OUTER_PRODUCT_ENUMERATION_ENTRY         *ThreadOuterProduct;
        D3D12_LINEAR_ALGEBRA_ATOMIC_ACCUMULATE_STORE_ENUMERATION_ENTRY      *AccumulateStore;
    };
} D3D12_FEATURE_DATA_LINEAR_ALGEBRA_OPERATION_ENUMERATION;
```

The intended usage is the standard two-call sequence: pass `NumEntries = 0` and a null union pointer to learn how many entries the driver would emit, allocate, then call again with the allocated array and `NumEntries` set to its capacity.

##### Enumeration Usage Example

```cpp
// Discover every native wave-scope matrix multiply configuration the driver supports.
D3D12_FEATURE_DATA_LINEAR_ALGEBRA_OPERATION_ENUMERATION enumerate = {};
enumerate.OperationType = D3D12_LINEAR_ALGEBRA_OPERATION_TYPE_WAVE_MATRIX_MULTIPLY;

// First call: get the count.
HRESULT hr = device->CheckFeatureSupport(
    D3D12_FEATURE_LINEAR_ALGEBRA_OPERATION_ENUMERATION,
    &enumerate,
    sizeof(enumerate));

if (SUCCEEDED(hr) && enumerate.NumEntries > 0)
{
    std::vector<D3D12_LINEAR_ALGEBRA_WAVE_MATRIX_MULTIPLY_ENUMERATION_ENTRY> entries(
        enumerate.NumEntries);
    enumerate.WaveMatrixMultiply = entries.data();

    // Second call: fill the array.
    hr = device->CheckFeatureSupport(
        D3D12_FEATURE_LINEAR_ALGEBRA_OPERATION_ENUMERATION,
        &enumerate,
        sizeof(enumerate));

    // entries[] now holds (MinWaveSize, MaxWaveSize, A, B, Acc, SupportFlags, Shape) tuples
    // and the application can pick the configuration that best fits its workload.
}
```


#### D3D12_LINEAR_ALGEBRA_TIER

The `D3D12_LINEAR_ALGEBRA_TIER` enumeration is the primary mechanism for standardizing linear algebra support across D3D12 hardware. By defining discrete tiers with mandatory format and dimension support, this system eliminates the need for developers to query and handle dozens of individual capability bits for different combinations of data types, matrix sizes, and operation scopes. Each tier represents a well-tested, cohesive set of capabilities that hardware vendors commit to supporting in their entirety, ensuring that applications can target a specific tier and rely on all associated features being available.

```cpp
typedef enum D3D12_LINEAR_ALGEBRA_TIER
{
    D3D12_LINEAR_ALGEBRA_TIER_NOT_SUPPORTED = 0,
    D3D12_LINEAR_ALGEBRA_TIER_1_0 = 0x10,
} D3D12_LINEAR_ALGEBRA_TIER;
```

**Members:**

- **D3D12_LINEAR_ALGEBRA_TIER_NOT_SUPPORTED** - The device does not support linear algebra operations.

- **D3D12_LINEAR_ALGEBRA_TIER_1_0** - The device supports Tier 1 linear algebra operations. See the [Tier 1 Support](#tier-1-support) section.

**Usage:**

Applications can query for linear algebra support using the feature check:

```cpp
D3D12_FEATURE_DATA_LINEAR_ALGEBRA_SUPPORT linearAlgebraSupport = {};
HRESULT hr = device->CheckFeatureSupport(
    D3D12_FEATURE_LINEAR_ALGEBRA_SUPPORT,
    &linearAlgebraSupport,
    sizeof(linearAlgebraSupport));

if (SUCCEEDED(hr) && linearAlgebraSupport.LinearAlgebraTier >= D3D12_LINEAR_ALGEBRA_TIER_1_0)
{
    // Device supports Tier 1 linear algebra operations
}
```
#### Tier 1 Support

As mentioned in the [introduction](#introduction), the primary capability being queried here is data types and tile shapes. Tier 1 devices must support:

##### Matrix-Matrix Operations

  A         |  B         |   Acc.  | Native   |
------------|------------|---------|----------|
  UInt8     |  UInt8     |  SInt32 | Required |
  SInt8     |  SInt8     |  SInt32 | Required |
  Fp16      |  Fp16      |  Fp16   | Optional |

**Column Definitions:**

- **A** and **B**: The data format for input matrices A and B. Tier 1 only requires support for matched A/B types; integer signedness is not required to match between A and B. A driver may expose mixed-signedness A/B combinations as an additional capability through the granular [wave-scope](#d3d12_linear_algebra_wave_matrix_multiply_support) and [threadgroup-scope](#d3d12_linear_algebra_threadgroup_matrix_multiply_support) queries, or discoverable via the [Operation Enumeration API](#operation-enumeration-api).
- **Acc.**: The accumulator format used for intermediate and final results. Higher precision accumulators prevent overflow and maintain accuracy during computation.

It is valid for drivers to use higher internal precision for Fp16 multiplication and then convert final results to Fp16.

##### Vector-Matrix Operations

Vector | Matrix   | Result | Native   |
-------|----------|--------|----------|
SInt8  | SInt8    | SInt32 | Required |
UInt8  | UInt8    | SInt32 | Required |
Fp32   | SInt8    | SInt32 | Required |
Fp16   | Fp16     | Fp16   | Required |
Fp16   | Fp8_E4M3 | Fp16   | Optional |
Fp16   | Fp8_E5M2 | Fp16   | Optional |

Column meaning, HLSL supply paths, and conversion behavior are described under [D3D12_LINEAR_ALGEBRA_THREAD_VECTOR_MATRIX_MULTIPLY_SUPPORT](#d3d12_linear_algebra_thread_vector_matrix_multiply_support). The **Native** column governs whether tier-1 implementations are required to accelerate the row natively:

* `Required` -- implementations MUST accept the row and multiply it natively (no `EMULATED_INPUTS`). `EMULATED_OUTPUTS` is permitted: as described for [`EMULATED_OUTPUTS`](#d3d12_linear_algebra_multiplication_support_flags) and for [Fp16 matrix-matrix multiplication](#matrix-matrix-operations), accumulating at a higher internal precision with a final conversion step is always allowed, so a `Required` row with a sub-32-bit result type may still report `EMULATED_OUTPUTS`.
* `Optional` -- implementations MUST accept the row but MAY emulate it, including the multiply itself; the granular query reports the emulation strategy via `EMULATED_INPUTS` and/or `EMULATED_OUTPUTS`.

Integer signedness is not required to match between the vector type and the matrix type. Every row in the table happens to list matched-signedness combinations because mixed-signedness support is not part of the tier-1 contract, but a driver may expose any mixed-signedness combination as an additional capability through the granular and enumeration queries.

Note: Bias is omitted from the table. It is required that bias types matching the result type must be supported, as well as `NONE` (no bias).

Note that FP8 data types are required to be supported for inputs, but it is recognized that these may not be natively supported. If these are not natively supported, the driver is required to emulate them. This emulation is required, recognizing that applications will not want to ship multiple versions of models, and if a model is quantized to FP8, applications should be able to ship the smallest version of that model. If FP8 was optional, it would increase the install footprint of applications leveraging this functionality.

Due to hardware diversity, both emulation of the conversion to FP8, as well as emulation of FP8->FP16 multiplication are allowed. Transposing loads and stores are not required for any format.

##### Native Matrix Dimensions

For wave-scope and threadgroup-scope multiplication, the driver natively supports one or more tile shapes per type combination, discoverable via the [Operation Enumeration API](#operation-enumeration-api). Application matrix shapes are accepted by the driver -- and reported as supported by the granular [wave-scope](#d3d12_linear_algebra_wave_matrix_multiply_support) and [threadgroup-scope](#d3d12_linear_algebra_threadgroup_matrix_multiply_support) queries -- when they are an integer multiple of one of those native shapes in each dimension; matrices smaller than the smallest native shape are not supported.

For a supported wave-scope or threadgroup-scope data type, there must be at least one reported shape whose *largest* component is less than or equal to 16 for types that are 16-bit or larger, or whose largest bit size is 256 for types that are smaller than 16-bit. This ensures that 16x16x16 will always be a valid tile shape for 16-bit types, while smaller types may support a minimum size of 32x16x16.

Applications that want to use smaller shapes will need to query them on a case-by-case basis. Hardware that wants to expose larger tile sizes must do so alongside a size that meets this requirement.

Note that there is no requirement around thread-scope vector-matrix multiplication dimensions.

##### Outer Product

No specific formats are required to be supported for tier 1.

##### Accumulation Store

Accumulation store requires atomic addition to memory, either for vectors or matrices. No formats are required to be supported for tier 1. Optional formats to support are FP16 and FP32.

#### Runtime Validation

The HLSL compiler emits detailed metadata about vector and matrix operations used in shaders as part of the Pipeline State Validation (PSV0) section of the shader bytecode. This metadata includes information about the specific linear algebra operations invoked, the data formats used (input, accumulator, and output types), matrix dimensions, and operation scopes (Thread, Wave, or ThreadGroup). When a shader containing linear algebra operations is used to create a pipeline state object, the D3D12 runtime parses this PSV0 metadata and validates it against the hardware's reported `D3D12_FEATURE_LINEAR_ALGEBRA_MATRIX_OPERATION_SUPPORT` capabilities. If the shader requests operations, formats, or dimensions that are not supported by the driver, the runtime will fail pipeline creation with a descriptive error indicating which specific capability requirement was not met.

#### Convert Matrix Layout

The linear algebra load, store, and accumulate intrinsics use
`ByteAddressBuffer` or `RWByteAddressBuffer` resources with
implementation-specific layouts and performance characteristics. The following
driver-side API converts a weight matrix between layouts in
`D3D12_LINEAR_ALGEBRA_MATRIX_LAYOUT`. It does not convert element data types.

```c++
enum D3D12_LINEAR_ALGEBRA_MATRIX_LAYOUT
{
    D3D12_LINEAR_ALGEBRA_MATRIX_LAYOUT_ROW_MAJOR,
    D3D12_LINEAR_ALGEBRA_MATRIX_LAYOUT_COLUMN_MAJOR,
    D3D12_LINEAR_ALGEBRA_MATRIX_LAYOUT_MUL_OPTIMAL,
    D3D12_LINEAR_ALGEBRA_MATRIX_LAYOUT_MUL_OPTIMAL_TRANSPOSE,
    D3D12_LINEAR_ALGEBRA_MATRIX_LAYOUT_OUTER_PRODUCT_OPTIMAL,
    D3D12_LINEAR_ALGEBRA_MATRIX_LAYOUT_OUTER_PRODUCT_OPTIMAL_TRANSPOSE,
  };
```

##### Query Destination Size

The destination buffer size can be implementation-dependent. Use
`GetLinearAlgebraMatrixConversionDestinationInfo` to query the size for the
destination layout and data type. The method updates `DestSize` in the supplied
`D3D12_LINEAR_ALGEBRA_MATRIX_CONVERSION_DEST_INFO`. Pass that updated descriptor
to the conversion method.

The `DestSize` and `DestStride` must be a multiple of 16 bytes. The `DestVA` must be 128-byte aligned.

```c++

// Descriptor to query the destination buffer size
typedef struct D3D12_LINEAR_ALGEBRA_MATRIX_CONVERSION_DEST_INFO { 
    UINT                                   DestSize;      // !< [out]Destination buffer size in bytes
                                                          // required for conversion 
    D3D12_LINEAR_ALGEBRA_MATRIX_LAYOUT     DestLayout;    // !< [in] Is the layout the matrix is converted to
    UINT                                   DestStride;    // !< [in] Is the number of bytes between a consecutive 
                                                          // row or column (depending on DestLayout) of the 
                                                          // destination matrix if it is row-major or 
                                                          // column-major.
    UINT                                   NumRows;       // !< [in] Is the number of rows in the matrix. 
    UINT                                   NumColumns;    // !< [in] Is the number of columns in the matrix. 
    D3D12_LINEAR_ALGEBRA_DATATYPE          DestDataType;  // !< [in] the type of a destination matrix element. 
};

// An API to return the number of bytes required in the destination buffer to
// store the result of conversion. The size of the destination is a function of
// the destination layout information and does not depend on the source layout
// information.

void ID3D12DevicePreview::GetLinearAlgebraMatrixConversionDestinationInfo(
    D3D12_LINEAR_ALGEBRA_MATRIX_CONVERSION_DEST_INFO* pDesc);

```

##### Conversion descriptors

After obtaining the destination size, the application supplies the destination
descriptor, source layout and data type in
`D3D12_LINEAR_ALGEBRA_MATRIX_CONVERSION_SRC_INFO`, and the source and
destination addresses to the layout conversion API.

```c++

// GPU VAs of source and destination buffers

typedef struct D3D12_LINEAR_ALGEBRA_MATRIX_CONVERSION_DATA {
    D3D12_GPU_VIRTUAL_ADDRESS               DestVA;               //!< [inout] GPU VA of destination 
                                                                  // buffer
    D3D12_GPU_VIRTUAL_ADDRESS               SrcVA;                //!< [in]    GPU VA of source 
                                                                  // buffer
};
 
// Source information descriptor. Destination information comes from 
// D3D12_LINEAR_ALGEBRA_MATRIX_CONVERSION_DEST_INFO

typedef struct D3D12_LINEAR_ALGEBRA_MATRIX_CONVERSION_SRC_INFO {
    UINT                                    SrcSize;                // !< [in] Is the length in bytes of 
                                                                    // srcData    
    D3D12_LINEAR_ALGEBRA_DATATYPE           SrcDataType;            // !< [in] Is the type of a 
                                                                    // source matrix 
                                                                    // element        
    D3D12_LINEAR_ALGEBRA_MATRIX_LAYOUT      SrcLayout;              // !< [in] Is the layout of the 
                                                                    // source matrix.
    UINT                                    SrcStride;              // !< [in] Is the number of bytes  
                                                                    // between a consecutive row or column 
                                                                    // (depending on srcLayout) 
                                                                    // of the source matrix, if it is row-major 
                                                                    // or column-major.   
};

// Descriptor passed to the conversion API
typedef struct D3D12_LINEAR_ALGEBRA_MATRIX_CONVERSION_INFO {
    D3D12_LINEAR_ALGEBRA_MATRIX_CONVERSION_DEST_INFO      DestInfo;
    D3D12_LINEAR_ALGEBRA_MATRIX_CONVERSION_SRC_INFO       SrcInfo;    
    D3D12_LINEAR_ALGEBRA_MATRIX_CONVERSION_DATA           DataDesc;   
};
```

##### Conversion APIs

New API is added to the ID3D12CommandList interface. Multiple conversions can be done in a single call of the API. The number of descriptors pointed to by pDesc is specified using DescCount. If DestSize passed to this API is less than the number of bytes returned in call to `GetLinearAlgebraMatrixConversionDestinationInfo`, behavior is undefined.

```c++
// Converts a source matrix to the destination layout.
void ID3D12GraphicsCommandListPreview::ConvertLinearAlgebraMatrix(
    D3D12_LINEAR_ALGEBRA_MATRIX_CONVERSION_INFO* pDesc,
    UINT DescCount);

```

*Valid Usage:* 

* If `SrcLayout` is row-major or column-major, `SrcStride` MUST be at least the
  length of a row or column and a multiple of the element size.
* If `DestLayout` is row-major or column-major, `DestStride` MUST be at least
  the length of a row or column and a multiple of 16.
* `SrcDataType` and `DestDataType` MUST be identical. This operation changes
  memory layout but does not convert data.
* `SrcDataType` and `DestDataType` MUST be valid for vector-matrix or
  matrix-matrix multiplication.
* The source and destination memory ranges MUST NOT overlap.

*CommandList interactions:*

- Synchronization around `ConvertLinearAlgebraMatrix` calls:
   - Legacy Barrier
     - Source buffer: Must be in `D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE` state
     - Dest buffer: Must be in `D3D12_RESOURCE_STATE_UNORDERED_ACCESS` state
     - UAV barrier synchronizes writes to the destination
   - Enhanced Barrier:
     - Source buffer access: `D3D12_BARRIER_ACCESS_SHADER_RESOURCE`
     - Dest buffer access: `D3D12_BARRIER_ACCESS_UNORDERED_ACCESS`
     - Sync point: `D3D12_BARRIER_SYNC_CONVERT_LINEAR_ALGEBRA_MATRIX`
 - Predication is supported
 - Available in Compute or Graphics CommandLists
 - Not supported in Bundles

*Usage Example:*

```c++

D3D12_LINEAR_ALGEBRA_MATRIX_CONVERSION_INFO infoDesc = 
{ 
    // DestInfo
    {
        0,                                                              // DestSize to be populated by 
                                                                        // driver implementation
        D3D12_LINEAR_ALGEBRA_MATRIX_LAYOUT_MUL_OPTIMAL,                 // convert to mul optimal layout
        0,                                                              // stride is ignored since optimal layout 
                                                                        // is implementation dependent
        numRows,                                                        // number of rows in weight matrix to be 
                                                                        // converted
        numColumns,                                                     // number of columns in weight matrix to 
                                                                        // be converted
        D3D12_LINEAR_ALGEBRA_DATATYPE_FLOAT8_E4M3FN                     // FP8 datatype
    },

    //SrcInfo
    {
        srcSize,                                                        // number of bytes of matrix in source 
                                                                        // layout and datatype
        D3D12_LINEAR_ALGEBRA_DATATYPE_FLOAT8_E4M3FN,                    // FP8 datatype
        D3D12_LINEAR_ALGEBRA_MATRIX_LAYOUT_ROW_MAJOR,                   // convert from row major layout
        (numColumns * sizeof(uint8_t))                                  // row major stride without padding
    },

    //DataDesc
    {
        0,                                                              // dest buffer address not known yet. 
                                                                        // Will be initialized after destSize
                                                                        // query
        srcVA                                                           // GPU VA of src buffer
    }                                              
                                    };

// Query destSize
pD3D12Device->GetLinearAlgebraMatrixConversionDestinationInfo(&infoDesc.DestInfo);

// destVA identifies a non-overlapping allocation aligned to 128 bytes.
infoDesc.DataDesc.DestVA = destVA;

// Perform the conversion
pD3D12CommandList->ConvertLinearAlgebraMatrix(&infoDesc, 1);

```

### DDI Changes

The linear algebra DDI is exposed under D3D12 core DDI version 0115, using
function table type `D3D12DDI_TABLE_TYPE_0115_LINEAR_ALGEBRA` and `D3D12DDI_FEATURE`
enum value 15 (shared with cooperative vectors). The feature version this spec
defines is `D3D12DDI_FEATURE_VERSION_LINEAR_ALGEBRA_0115_2`; drivers reporting
this feature version MUST implement every requirement described below.

#### Granular caps query

`D3D12DDICAPS_TYPE_LINEAR_ALGEBRA_MATRIX_OPERATION_SUPPORT` is the per-configuration
"is this specific configuration supported?" caps query. Its data struct
`D3D12DDI_LINEAR_ALGEBRA_MATRIX_OPERATION_SUPPORT_0115_2` mirrors the API
[D3D12_FEATURE_DATA_LINEAR_ALGEBRA_MATRIX_OPERATION_SUPPORT](#d3d12_feature_data_linear_algebra_matrix_operation_support)
field-for-field, with each per-op-type DDI struct mirroring its API counterpart
(single-shape input, single-result output). Drivers MAY implement this caps
query on a per-operation-type basis; see Per-op-type query form advertisement
below.

#### Enumeration caps query

`D3D12DDICAPS_TYPE_LINEAR_ALGEBRA_OPERATION_ENUMERATION` is the
"give me every supported configuration for this operation type" caps query. Its
data struct `D3D12DDI_LINEAR_ALGEBRA_OPERATION_ENUMERATION_0115_2` mirrors the
API [D3D12_FEATURE_DATA_LINEAR_ALGEBRA_OPERATION_ENUMERATION](#d3d12_feature_data_linear_algebra_operation_enumeration)
field-for-field. The union holds DDI-namespace pointer types
(`D3D12DDI_LINEAR_ALGEBRA_*_ENUMERATION_ENTRY_0115_2 *`) that mirror their API
counterparts. The runtime services the caps query with the standard two-call
sequence (size query, then array fill); the driver MUST return the same
`NumEntries` value across the two calls for a given operation type.

The enumerated table is required to be stable for the lifetime of the device
and independent of any per-process or per-pipeline state. The runtime caches
the table at device init and uses it both to serve application enumeration
queries and to synthesize per-configuration answers for operation types where
the driver does not implement the granular caps query.

Drivers MUST implement this caps query for every operation type their device
supports.

#### Per-op-type query form advertisement

Drivers advertise which operation types they serve granularly via a caps query:

```c
typedef enum D3D12DDI_LINEAR_ALGEBRA_QUERY_FORM_FLAGS_0115_2
{
    D3D12DDI_LINEAR_ALGEBRA_QUERY_FORM_FLAG_NONE        = 0x0,
    D3D12DDI_LINEAR_ALGEBRA_QUERY_FORM_FLAG_GRANULAR    = 0x1,
    D3D12DDI_LINEAR_ALGEBRA_QUERY_FORM_FLAG_ENUMERATION = 0x2,
} D3D12DDI_LINEAR_ALGEBRA_QUERY_FORM_FLAGS_0115_2;

typedef struct D3D12DDI_LINEAR_ALGEBRA_QUERY_FORMS_0115_2
{
    D3D12DDI_LINEAR_ALGEBRA_QUERY_FORM_FLAGS_0115_2 MatrixConstruction;
    D3D12DDI_LINEAR_ALGEBRA_QUERY_FORM_FLAGS_0115_2 WaveMatrixMultiply;
    D3D12DDI_LINEAR_ALGEBRA_QUERY_FORM_FLAGS_0115_2 ThreadGroupMatrixMultiply;
    D3D12DDI_LINEAR_ALGEBRA_QUERY_FORM_FLAGS_0115_2 ThreadVectorMatrixMultiply;
    D3D12DDI_LINEAR_ALGEBRA_QUERY_FORM_FLAGS_0115_2 ThreadOuterProduct;
    D3D12DDI_LINEAR_ALGEBRA_QUERY_FORM_FLAGS_0115_2 AtomicAccumulateStore;
} D3D12DDI_LINEAR_ALGEBRA_QUERY_FORMS_0115_2;
```

Queried via caps type `D3D12DDICAPS_TYPE_LINEAR_ALGEBRA_QUERY_FORMS` once at
device init. Required behavior:

* Each field MUST include `ENUMERATION` for operation types the device supports
  and MUST be `NONE` for operation types the device does not support.
* `GRANULAR` is set on a field when the driver implements the granular caps
  handler for that operation type. The runtime forwards application granular
  queries to the driver when this flag is set, and synthesizes answers from
  the cached enumeration table otherwise. Drivers without efficient
  predicate-based granular logic should leave this flag clear.
* Reported form support is fixed for the lifetime of the device.

When both forms are advertised for an operation type, granular and enumeration
results MUST be self-consistent: for every `(types, shape)` combination the
enumeration would expand to (including every integer-multiple shape of any
native shape), the granular query MUST report supported, and vice versa. The
D3D12 debug layer cross-checks this invariant on every application-level
granular call when both forms are advertised.

#### `_1` deprecation

The prior preview release advertised `D3D12DDI_FEATURE_VERSION_LINEAR_ALGEBRA_0115_1`,
with a different granular caps struct shape -- `MATRIX_CONSTRUCTION_SUPPORT`
returned a single `MinM/MinK/MinN` triple, and `WAVE_MATRIX_MULTIPLY_SUPPORT`
returned a native-shape array -- and no enumeration caps query. `_1` is
deprecated; new drivers SHOULD report `_2` exclusively.

The runtime continues to load `_1` drivers and exposes the granular
[API](#granular-capability-query-api) on them by translating each `_2`-shaped
application query into the corresponding `_1` DDI call(s):

* `MATRIX_CONSTRUCTION_SUPPORT`: runtime reads `MinM/MinK/MinN` from the `_1`
  DDI and returns `Supported = TRUE` iff the application's `Shape` is an integer
  multiple of `(MinM, MinK, MinN)` in each dimension. (The `_1` DDI only reports
  a single tiling, so drivers using `_1` cannot express multi-tiling support;
  this is a `_1` limitation, not a runtime translation issue.)
* `WAVE_MATRIX_MULTIPLY_SUPPORT`: runtime reads the native shape array from the
  `_1` DDI and returns supported iff the application's `Shape` is an integer
  multiple of any reported native shape in each dimension.
* `THREADGROUP_MATRIX_MULTIPLY_SUPPORT`: shape input was already present in
  `_1`; the runtime forwards the query directly.
* Other operation types: `_1` and `_2` granular shapes are identical; the
  runtime forwards directly.

The [Operation Enumeration API](#operation-enumeration-api) is not exposed on
`_1` drivers -- `CheckFeatureSupport` for
`D3D12_FEATURE_LINEAR_ALGEBRA_OPERATION_ENUMERATION` returns
`DXGI_ERROR_UNSUPPORTED` on a device backed by a `_1` driver, regardless of
operation type. Applications that depend on enumeration must fall back to the
granular query or report an unsupported configuration to the user.

### WARP Support

WARP must implement the Tier 1 capability set, the linear algebra DXIL
operations accepted for that tier, runtime validation, and matrix layout
conversion behavior. The implementation and performance strategy remain to be
specified before this proposal advances to Review.

### PIX Support

PIX must recognize the new HLSL and DXIL operations, decode linear algebra PSV
metadata, and present matrix operation types, shapes, scopes, and component
types in shader and pipeline inspection. Capture and debugging requirements
remain to be specified before Review.

### Other Tooling Impact

DXC, the DXIL validator, the D3D12 debug layer, SDK headers, and conformance
infrastructure require coordinated updates. SPIR-V support is limited to the
portable subset described by the HLSL API and the relevant cooperative-matrix
extensions.

### HLSL Additions

#### HLSL API Concepts

The new HLSL API introduces a new `linalg::Matrix` type which represents an
opaque matrix object, and contains an intangible value object that refers to the
matrix.

The `linalg::Matrix` template type is parameterized based on the matrix
component data type, dimensions, use, and scope. These parameters restrict where
and how a matrix can be used.

#### Stage Availability

All operations on `Thread` scope matrices are available in all shader stages.
Operations on `Wave` and `ThreadGroup` scope matrices are available in compute
shaders.

##### Matrix Use

The `Use` parameter of an instance of a `linalg::Matrix` denotes which argument
it can be in matrix-matrix operations or matrix-vector operations.

There are three matrix usages: `A`, `B`, and `Accumulator`.
* The `A` matrix usage denotes a matrix that can be the first argument to binary
  or ternary algebraic operations.
* The `B` matrix usage denotes a matrix that can be the second argument to binary
  or ternary algebraic operations.
* The `Accumulator` matrix usage denotes a matrix that can either be a produced
  result from a binary arithmetic operation, or the third argument to a ternary
  algebraic operation.

The matrix use type parameter enables implementations to optimize the storage
and layout of the matrix prior to tensor operations. It may be expensive on some
hardware to translate between matrix uses, for that reason we capture the use in
the type and require explicit conversion in the HLSL API.

Throughout this document a matrix may be described as a matrix of its use (e.g.
a matrix with `Use == Accumulator` is an _accumulator matrix_, while a matrix
with use `A` is an _A matrix_.)

##### Matrix Scope

The `Scope` parameter of an instance of a `linalg::Matrix` denotes the
uniformity scope of the matrix. The scope impacts which operations can be
performed on the matrix and may have performance implications depending on the
implementation.

There are three supported matrix scopes: `Thread`, `Wave`, and `ThreadGroup`.
* The `Thread` matrix scope denotes that a matrix's values may vary by thread,
  which requires that an implementation handle divergent matrix values.
* The `Wave` matrix scope denotes that a matrix's values are uniform across a
  wave, which allows an implementation to assume all instances of the matrix
  across a wave are identical.
* The `ThreadGroup` matrix scope denotes that a matrix's values are uniform
  across a thread group, which allows an implementation to assume all instances
  of the matrix across a thread group are identical.

Operations are categorized by their scope requirements. Some operations require
uniform scope matrices (`Wave` or`ThreadGroup`), while others can operate on
non-uniform (`Thread`) scope matrices. Operations must be called from HLSL under
control flow that is _at least_ as uniform as the matrix scope. `Thread`-scope
may be called in non-uniform control flow, `Wave`-scope operations must be
called in `Wave`-uniform control flow, and `ThreadGroup`-scope operations must
be called in `ThreadGroup`-uniform control flow. Operations implicitly
synchronize execution across all threads in the matrix's scope. Calling an
operation from control flow that is not uniform across all participating threads
is undefined behavior.

When using `ThreadGroup` scope matrices, explicit barriers are required only when
there are actual cross-thread dependencies, such as when multiple threads
contribute to building or modifying the matrix before it is used by other
threads. The matrix scope semantics handle most synchronization automatically,
eliminating the need for barriers between every matrix operation.

The following table summarizes the operations supported for each matrix scope:

| Operation | Thread Scope | Wave Scope | ThreadGroup Scope |
|-----------|--------------|------------|-------------------|
| `Matrix::Cast()` | ✗ | ✓ | ✓ |
| `Matrix::Length()` | ✗ | ✓ | ✓ |
| `Matrix::GetCoordinate(uint)` | ✗ | ✓ | ✓ |
| `Matrix::Get(uint)` | ✗ | ✓ | ✓ |
| `Matrix::Set(uint, T)` | ✗ | ✓ | ✓ |
| `Matrix::Splat()` | ✗ | ✓ | ✓ |
| `Matrix::Load(ByteAddressBuffer)` | ✓ | ✓ | ✓ |
| `Matrix::Load(RWByteAddressBuffer)` | ✗ | ✓ | ✓ |
| `Matrix::Load(groupshared)` | ✗ | ✓ | ✓ |
| `Matrix::Store(RWByteAddressBuffer)` | ✗ | ✓ | ✓ |
| `Matrix::Store(groupshared)` | ✗ | ✓ | ✓ |
| `Matrix::InterlockedAccumulate(RWByteAddressBuffer)` | ✓ | ✓ | ✓ |
| `Matrix::InterlockedAccumulate(groupshared)` | ✗ | ✓ | ✓ |
| `Matrix::Accumulate(Matrix)` | ✗ | ✓ | ✓ |
| `Matrix::MultiplyAccumulate()` | ✗ | ✓ | ✓ |
| `linalg::Multiply(Matrix, Matrix)` | ✗ | ✓ | ✓ |
| `linalg::Multiply(Matrix, vector)` | ✓ | ✗ | ✗ |
| `linalg::MultiplyAdd(Matrix, vector, vector)` | ✓ | ✗ | ✗ |
| `linalg::OuterProduct(vector, vector)` | ✓ | ✗ | ✗ |
| `linalg::InterlockedAccumulate(RWByteAddressBuffer, vector)` | ✓ | ✗ | ✗ |

Throughout this document a matrix may be described as having a scope as
specified by the `Scope` parameter (e.g. a matrix with `Scope == Thread` is a
_matrix with thread scope_).

Matrix storage is always opaque, the `Scope` does not directly restrict how the
matrix is stored, it merely denotes allowed scopes of allowed data divergence.
A matrix with thread scope must behave as if each thread has a unique copy of
the matrix. An implementation may coalesce identical matrices across threads.

##### Matrix Storage

In HLSL, matrix objects are intangible objects so they do not have defined size
or memory layout. When in use, implementations are expected to distribute the
storage of matrices across the thread-local storage for all threads in a SIMD
unit. An implementation may also utilize caches or other memory regions as
appropriate. At the DXIL level a matrix is represented as a value object.
Because LLVM 3.7 doesn't allow value objects of opaque types, the matrix object
stores a pointer in the IR, but implementations will replace this with an
implementation-defined object.

An A matrix is a collection of per-thread vectors representing matrix rows,
while a B matrix is a collection of per-thread vectors representing matrix
columns.

An Accumulator matrix may be either an A matrix, or a B matrix, and it varies by
hardware implementation.

##### Restrictions on Dimensions

The HLSL API will enforce restrictions on the `K` dimension as found in the
formula: `MxK * KxN = MxN`

This restriction applies to the number of columns in an A matrix and the number
of rows in a B matrix. It does not apply to an accumulator matrix.

The minimum and maximum `K` dimension for matrices is hardware dependent and
varies by scope. The table below gives the inclusive ranges enforced by HLSL
and DXIL validation.


| Matrix scope | Inclusive `K` range |
| ------------ | ------------------- |
| `Thread`     | [4, 128]            |
| `Wave`       | [4, 128]            |
| `ThreadGroup`| [1, 1024]           |

Not all hardware is required to support all possible dimensions for thread and
wave scope matrices, or all possible element types. The shader compiler will
encode the dimensions and input and output data types used by each shader in the
[Pipeline State Validation metadata](#pipeline-state-validation-metadata).

#### HLSL API Documentation

##### Summary

The `dx::linalg` namespace provides:

* `ComponentType`, `MatrixUse`, `MatrixScope`, and `MatrixLayout` enumerations;
* `VectorRef`, `InterpretedVector`, and `Matrix` class templates;
* matrix construction, conversion, inspection, arithmetic, and memory
  operations; and
* vector conversion, matrix-vector arithmetic, outer product, and atomic
  accumulation functions.

This feature also adds `hlsl::enable_if` and the type traits required to
constrain these declarations.

##### Detailed specifications

###### HLSL Enumerations
```c++
// Put this in a dxil constants header.
namespace dxil {

// This enum must _exactly_ match the DXIL constants.
enum class ComponentType : uint32_t {
  Invalid = 0,
  I1 = 1,
  I16 = 2,
  U16 = 3,
  I32 = 4,
  U32 = 5,
  I64 = 6,
  U64 = 7,
  F16 = 8,
  F32 = 9,
  F64 = 10,
  SNormF16 = 11,
  UNormF16 = 12,
  SNormF32 = 13,
  UNormF32 = 14,
  SNormF64 = 15,
  UNormF64 = 16,
  PackedS8x32 = 17,
  PackedU8x32 = 18,

  // BEGIN NEW FOR SM 6.10
  I8 = 19,
  U8 = 20,
  F8_E4M3FN = 21,
  F8_E5M2 = 22,
  BFloat16 = 23,
  // END

  LastEntry
};

} // namespace dxil

namespace dx {

namespace linalg {

#define __COMPONENT_TYPE(type) type = (uint)dxil::ComponentType::type

// This enum only defines values that are valid for Matrix component types.
// Each enumeration's value matches the corresponding DXIL constant.
struct ComponentType {
  enum ComponentEnum {
    // Signed integers.
    __COMPONENT_TYPE(I8),
    __COMPONENT_TYPE(I16),
    __COMPONENT_TYPE(I32),
    __COMPONENT_TYPE(I64),

    // Unsigned integers.
    __COMPONENT_TYPE(U8),
    __COMPONENT_TYPE(U16),
    __COMPONENT_TYPE(U32),
    __COMPONENT_TYPE(U64),

    // Floating point types.
    __COMPONENT_TYPE(F8_E4M3FN),
    __COMPONENT_TYPE(F8_E5M2),
    __COMPONENT_TYPE(F16),
    __COMPONENT_TYPE(F32),
    __COMPONENT_TYPE(F64),
    __COMPONENT_TYPE(BFloat16),
  };
};

#undef __COMPONENT_TYPE

using ComponentEnum = ComponentType::ComponentEnum;

struct MatrixUse {
  enum MatrixUseEnum {
    A = 0,
    B = 1,
    Accumulator = 2,
  };
};
using MatrixUseEnum = MatrixUse::MatrixUseEnum;

struct MatrixScope {
  enum MatrixScopeEnum {
    Thread = 0,
    Wave = 1,
    ThreadGroup = 2,
  };
};
using MatrixScopeEnum = MatrixScope::MatrixScopeEnum;

struct MatrixLayout {
  enum MatrixLayoutEnum {
    RowMajor = 0,
    ColMajor = 1,
    MulOptimal = 2,
    MulOptimalTranspose = 3,
    OuterProductOptimal = 4,
    OuterProductOptimalTranspose = 5,
  };
};
using MatrixLayoutEnum = MatrixLayout::MatrixLayoutEnum;
```

`ComponentEnum` identifies matrix component formats. Its values equal the
corresponding DXIL component-type values. `MatrixUseEnum` identifies operand or
accumulator use. `MatrixScopeEnum` identifies the required uniformity scope.
`MatrixLayoutEnum` identifies an external matrix layout.

###### `hlsl::enable_if`

```c++
namespace hlsl {
template <bool B, typename T> struct enable_if {};

template <typename T> struct enable_if<true, T> {
  using type = T;
};

} // namespace hlsl
```

`hlsl::enable_if<B, T>` contains the member type `type`, equal to `T`, only when
`B` is `true`.

###### HLSL type traits

```c++
namespace hlsl {

#ifdef __hlsl_dx_compiler
#define SIZE_TYPE int
#else
#define SIZE_TYPE uint
#endif

template <typename T, typename U> struct is_same {
  static const bool value = false;
};

template <typename T> struct is_same<T, T> {
  static const bool value = true;
};

template <typename T> struct is_arithmetic {
  static const bool value = false;
};

#define __ARITHMETIC_TYPE(type)                                                \
  template <> struct is_arithmetic<type> {                                     \
    static const bool value = true;                                            \
  };

#if __HLSL_ENABLE_16_BIT
__ARITHMETIC_TYPE(uint16_t)
__ARITHMETIC_TYPE(int16_t)
#endif
__ARITHMETIC_TYPE(uint)
__ARITHMETIC_TYPE(int)
__ARITHMETIC_TYPE(uint64_t)
__ARITHMETIC_TYPE(int64_t)
__ARITHMETIC_TYPE(half)
__ARITHMETIC_TYPE(float)
__ARITHMETIC_TYPE(double)

} // namespace hlsl
```

`is_same<T, U>::value` is `true` if `T` and `U` are the same type.
`is_arithmetic<T>::value` is `true` for the listed scalar arithmetic types.

###### `dx::linalg::__detail` type traits

```c++
namespace __detail {
template <ComponentEnum CompTy> struct ComponentTypeTraits {
  using Type = uint;
  static const bool IsNativeScalar = false;
  static const uint ElementsPerScalar = 4;
};

template <typename T> struct TypeTraits {
  static const ComponentEnum CompType =
      (ComponentEnum)dxil::ComponentType::Invalid;
};

template<> struct ComponentTypeTraits<ComponentType::BFloat16> {
  using Type = uint;
  static const bool IsNativeScalar = false;
  static const uint ElementsPerScalar = 2;
};

#define __MATRIX_SCALAR_COMPONENT_MAPPING(enum_val, type)                      \
  template <> struct ComponentTypeTraits<enum_val> {                           \
    using Type = type;                                                         \
    static const bool IsNativeScalar = true;                                   \
    static const uint ElementsPerScalar = 1;                                   \
  };                                                                           \
  template <> struct TypeTraits<type> {                                        \
    static const ComponentEnum CompType = enum_val;                            \
  };

#if __HLSL_ENABLE_16_BIT
__MATRIX_SCALAR_COMPONENT_MAPPING(ComponentType::I16, int16_t)
__MATRIX_SCALAR_COMPONENT_MAPPING(ComponentType::U16, uint16_t)
__MATRIX_SCALAR_COMPONENT_MAPPING(ComponentType::F16, float16_t)
#endif

__MATRIX_SCALAR_COMPONENT_MAPPING(ComponentType::I32, int32_t)
__MATRIX_SCALAR_COMPONENT_MAPPING(ComponentType::U32, uint32_t)
__MATRIX_SCALAR_COMPONENT_MAPPING(ComponentType::F32, float)
__MATRIX_SCALAR_COMPONENT_MAPPING(ComponentType::I64, int64_t)
__MATRIX_SCALAR_COMPONENT_MAPPING(ComponentType::U64, uint64_t)
__MATRIX_SCALAR_COMPONENT_MAPPING(ComponentType::F64, double)

template <ComponentEnum DstTy, ComponentEnum SrcTy, int SrcN> struct DstN {
  // Make sure to round up in case SrcN isn't an even multiple of the number of
  // elements per scalar
  static const int Value =
      (SrcN * ComponentTypeTraits<SrcTy>::ElementsPerScalar +
       ComponentTypeTraits<DstTy>::ElementsPerScalar - 1) /
      ComponentTypeTraits<DstTy>::ElementsPerScalar;
};

template <SIZE_TYPE MVal, SIZE_TYPE NVal, bool Transposed> struct DimMN {
  static const SIZE_TYPE M = MVal;
  static const SIZE_TYPE N = NVal;
};

template <SIZE_TYPE MVal, SIZE_TYPE NVal> struct DimMN<MVal, NVal, true> {
  static const SIZE_TYPE M = NVal;
  static const SIZE_TYPE N = MVal;
};

template <ComponentEnum CompTy, SIZE_TYPE PackedComponentCount>
struct ScalarCountFromPackedComponents {
  static const SIZE_TYPE ElementsPerScalar =
      ComponentTypeTraits<CompTy>::ElementsPerScalar;
  static const SIZE_TYPE Value =
      (PackedComponentCount + ElementsPerScalar - 1) / ElementsPerScalar;
};

template <ComponentEnum ElementType, SIZE_TYPE M, SIZE_TYPE N>
struct DefaultAlign {
  enum {
    MinDim = M < N ? M : N,
    ScalarCount = ScalarCountFromPackedComponents<ElementType, MinDim>::Value,
    ByteAlign = ScalarCount * 4,
    MinByteAlign = ByteAlign < 4 ? 4 : ByteAlign,
    Value = MinByteAlign < 16 ? MinByteAlign : 16
  };
};

} // namespace __detail
```

`ComponentTypeTraits` maps a component format to its storage type and packing
properties. `TypeTraits` performs the inverse mapping for native scalar types.
`DstN` computes the destination vector size for a packed conversion. `DimMN`
conditionally transposes two dimensions. `ScalarCountFromPackedComponents`
computes the scalar count for a packed component count. `DefaultAlign`
computes the default matrix alignment.

###### `VectorRef`

```c++
template <ComponentEnum ElementType, uint DimA>
struct VectorRef {
  ByteAddressBuffer Buf;
  uint Offset;
};
```

`VectorRef` identifies a `DimA`-element vector in `Buf` with component format
`ElementType`, beginning at byte offset `Offset`.

###### `InterpretedVector`

```c++
template <typename T, int N, ComponentEnum DT>
struct InterpretedVector {
  vector<T, N> Data;
  static const ComponentEnum Interpretation = DT;
  static const SIZE_TYPE Size =
      __detail::ComponentTypeTraits<DT>::ElementsPerScalar * N;
};
```

`InterpretedVector` associates the storage vector `Data` with component format
`DT`. `Size` is the number of interpreted components.

###### `Matrix`

```c++
template <ComponentEnum ComponentTy, SIZE_TYPE M, SIZE_TYPE N,
          MatrixUseEnum Use, MatrixScopeEnum Scope>
class Matrix;
```

`Matrix` is an opaque $M \times N$ matrix with component format `ComponentTy`,
use `Use`, and uniformity scope `Scope`. It has no defined object size or memory
layout. Its member functions are specified below.

###### `MakeInterpretedVector`

```c++
template <ComponentEnum DT, typename T, int N>
InterpretedVector<T, N, DT>
MakeInterpretedVector(vector<T, N> Vec);
```

- **Effects:** Associates `Vec` with component format `DT`. No conversion is
  performed.
- **Returns:** An `InterpretedVector<T, N, DT>` containing `Vec`.

###### `linalg::Convert`

```c++
template <ComponentEnum DestTy, ComponentEnum OriginTy, typename T, int N>
typename hlsl::enable_if<
    DestTy != OriginTy,
    InterpretedVector<typename __detail::ComponentTypeTraits<DestTy>::Type,
                      __detail::DstN<DestTy, OriginTy, N>::Value, DestTy>>::type
linalg::Convert(vector<T, N> Vec);

template <ComponentEnum DestTy, ComponentEnum OriginTy, typename T, int N>
typename hlsl::enable_if<DestTy == OriginTy,
                         InterpretedVector<T, N, DestTy>>::type
linalg::Convert(vector<T, N> Vec)
```

- **Requires:** If `OriginTy` has a native HLSL type, `T` is that type.
- **Effects:** Converts `Vec` from the `OriginTy` interpretation to `DestTy`
  according to [Data Conversion Rules](#data-conversion-rules).
- **Returns:** An `InterpretedVector` with interpretation `DestTy`.
- **Remarks:** These conversions differ from standard HLSL conversions.

###### `Matrix::Cast`

```c++
template <ComponentEnum NewCompTy, MatrixUseEnum NewUse = Use,
          bool Transpose = false>
Matrix<NewCompTy, __detail::DimMN<M, N, Transpose>::M,
       __detail::DimMN<M, N, Transpose>::N, NewUse, Scope>
Matrix::Cast();
```

- **Requires:** `Scope` is `Wave` or `ThreadGroup`.
- **Effects:** Converts the component type to `NewCompTy`, the use to `NewUse`,
  and transposes the matrix if `Transpose` is `true`.
- **Synchronization:** The call occurs in control flow uniform at `Scope`.
- **Returns:** The converted matrix. The dimensions are transposed if
  `Transpose` is `true`.

###### `Matrix::Splat`


```c++
template <typename T>
[[nodiscard]] static
    typename hlsl::enable_if<hlsl::is_arithmetic<T>::value, Matrix>::type
    Matrix::Splat(T Val);
```

- **Requires:** `Scope` is `Wave` or `ThreadGroup`.
- **Effects:** Initializes every matrix element from the first active lane's
  value of `Val`, converted to `ElementType`.
- **Synchronization:** The call occurs in control flow uniform at `Scope`.
- **Returns:** The initialized matrix.
- **Remarks:** The operation is equivalent to
  `Matrix::Splat(WaveReadLaneFirst(Val))`.

###### `Matrix::Load`

```c++
template <uint Align> // defaults 128 on thread-scope, 16 on other scopes.
static Matrix Matrix::Load(ByteAddressBuffer Res, uint StartOffset, uint Stride,
                           MatrixLayoutEnum Layout);

// Not available on Thread scope matrices.
template <uint Align>
static Matrix Matrix::Load(RWByteAddressBuffer Res, uint StartOffset,
                           uint Stride, MatrixLayoutEnum Layout);

// Not available on Thread scope matrices.
template <typename T, SIZE_TYPE Size>
static typename hlsl::enable_if<
    (hlsl::is_same<typename hlsl::strip_vector_type<T>::type,
                   ElementType>::value ||
     hlsl::is_integral_or_packed_integral<
         typename hlsl::strip_vector_type<T>::type>::value),
    Matrix>::type
Matrix::Load(groupshared T Arr[Size], uint StartIdx, uint Stride,
             MatrixLayoutEnum Layout);
```

- **Requires:** `Layout` and `Scope` form a valid combination:

| Operation                         | Matrix Scope          | Matrix Layout          |
|-----------------------------------|-----------------------|------------------------|
| `Matrix::Load(ByteAddressBuffer)` | `Thread`              | any                    |
| `Matrix::Load(*)`                 | `Wave`, `ThreadGroup` | `RowMajor`, `ColMajor` |

- **Requires:** The first matrix element is aligned to 128 bytes for `Thread` scope and 4 bytes otherwise.
- **Requires:** `Stride` is in bytes and is a multiple of 16 for `Thread` scope and 4 bytes otherwise.
- **Requires:** For the `groupshared` overload, `T` is `ElementType`, an integral type, or a
  packed integral type.
- **Remarks:** For buffer overloads, `StartOffset` is in bytes.
- **Remarks:** `StartIdx` and `Stride` count `ElementType` elements.
- **Effects:** Reads the matrix elements from `Res` or `Arr` in `Layout` order.
  No data conversion is performed.
- **Synchronization:** For `Wave` and `ThreadGroup` scope, the call occurs in
  control flow uniform at `Scope`. The operation performs no memory
  synchronization and its reads are not atomic.
- **Returns:** The loaded matrix.
- **Remarks:** Thread-scope transposed loads are supported only when reported by
  the device.
- **Remarks:** These are the intrinsic's minimum HLSL alignment requirements.
  Resources used with D3D12 must also satisfy the stricter requirements in
  [Resource Alignment Requirements](#resource-alignment-requirements).

###### `Matrix::Length`

```c++
uint Matrix::Length();
```

- **Requires:** `Scope` is `Wave` or `ThreadGroup`.
- **Requires:** `ComponentTy` has a native HLSL representation.
- **Returns:** The number of matrix elements accessible to the current thread.
- **Remarks:** The thread-to-element mapping is implementation-defined. Values
  may differ between threads, and multiple threads may access the same element.
  The sum across the scope is at least $M \times N$.
- **Notes:** The function may be called from non-uniform control flow, but code
  that assumes a particular per-thread distribution is not portable.

###### `Matrix::GetCoordinate`

```c++
uint2 Matrix::GetCoordinate(uint Index);
```

- **Requires:** `Scope` is `Wave` or `ThreadGroup`.
- **Requires:** `ComponentTy` has a native HLSL representation.
- **Returns:** For `Index < Length()`, the implementation-defined matrix
  coordinate for `Index`, with row in `x` and column in `y`. Otherwise, returns
  `uint2(0xFFFFFFFFu, 0xFFFFFFFFu)`.

###### `Matrix::Get`

```c++
ElementType Matrix::Get(uint Index);
```

- **Requires:** `Scope` is `Wave` or `ThreadGroup`.
- **Requires:** `ComponentTy` has a native HLSL representation.
- **Returns:** The element at thread-local `Index` when `Index < Length()`;
  otherwise, zero converted to `ElementType`.

###### `Matrix::Set`

```c++
void Matrix::Set(uint Index, ElementType Value);
```

- **Requires:** `Scope` is `Wave` or `ThreadGroup`.
- **Requires:** `ComponentTy` has a native HLSL representation.
- **Effects:** If `Index < Length()`, assigns `Value` to the element at
  thread-local `Index`. Otherwise, has no effect.

###### `Matrix::Store`

```c++
template <uint Align>
void Matrix::Store(RWByteAddressBuffer Res, uint StartOffset, uint Stride,
                   MatrixLayoutEnum Layout);

template <typename T, SIZE_TYPE Size>
static typename hlsl::enable_if<
    (hlsl::is_same<typename hlsl::strip_vector_type<T>::type,
                   ElementType>::value ||
     hlsl::is_integral_or_packed_integral<
         typename hlsl::strip_vector_type<T>::type>::value),
    void>::type
Matrix::Store(groupshared T Arr[Size], uint StartIdx, uint Stride,
              MatrixLayoutEnum Layout);
```

- **Requires:** `Scope` is `Wave` or `ThreadGroup`.
- **Requires:** `Layout` is `RowMajor` or `ColMajor`.
- **Requires:** For the buffer overload, the first matrix element and `Stride`
  are aligned to 4 bytes.
- **Requires:** For the `groupshared` overload, `T` is `ElementType`, an
  integral type, or a packed integral type.
- **Remarks:** For the buffer overload, `StartOffset` and `Stride` are in bytes.
- **Remarks:** For the `groupshared` overload, `StartIdx` and `Stride` count
  `ElementType` elements.
- **Effects:** Writes the matrix elements to `Res` or `Arr` in `Layout` order.
  No data conversion is performed.
- **Synchronization:** The call occurs in control flow uniform at `Scope`. The
  operation performs no memory synchronization and its writes are not atomic.

###### `Matrix::InterlockedAccumulate`

```c++

// When Scope != Thread, the following overloads are available:
template <uint Align, MatrixUseEnum UseLocal = Use>
typename hlsl::enable_if<Use == MatrixUse::Accumulator && UseLocal == Use,
                         void>::type
Matrix::InterlockedAccumulate(RWByteAddressBuffer Res, uint StartOffset,
                              uint Stride, MatrixLayoutEnum Layout);

template <typename T, MatrixUseEnum UseLocal = Use, SIZE_TYPE Size>
typename hlsl::enable_if<
    hlsl::is_same<typename hlsl::strip_vector_type<T>::type,
                  ElementType>::value &&
        hlsl::is_arithmetic_vector<T>::value && Use == MatrixUse::Accumulator &&
        UseLocal == Use,
    void>::type
Matrix::InterlockedAccumulate(groupshared T Arr[Size], uint StartIdx,
                              uint Stride, MatrixLayoutEnum Layout);

template <typename T, MatrixUseEnum UseLocal = Use, SIZE_TYPE Size>
typename hlsl::enable_if<
    hlsl::is_same<typename hlsl::strip_vector_type<T>::type,
                  uint8_t4_packed>::value &&
        Use == MatrixUse::Accumulator && UseLocal == Use,
    void>::type
Matrix::InterlockedAccumulate(groupshared T Arr[Size], uint StartIdx,
                              uint Stride, MatrixLayoutEnum Layout);

// When Scope == Thread, the following overload is available:
template <uint Align = 128, MatrixUseEnum UseLocal = Use>
typename hlsl::enable_if<Use == MatrixUse::Accumulator && UseLocal == Use,
                         void>::type
Matrix::InterlockedAccumulate(RWByteAddressBuffer Res, uint StartOffset);
```

- **Requires:** `Use` is `Accumulator`.
- **Requires:** For `Thread` scope, the destination is an
  `RWByteAddressBuffer`, its first matrix element is aligned to 128 bytes, and
  its layout is `OuterProductOptimal`.
- **Requires:** For `Wave` and `ThreadGroup` scope, `Layout` is `RowMajor` or
  `ColMajor`, and the first matrix element is aligned to 4 bytes.
- **Requires:** A `groupshared` destination has a permitted element type.
- **Effects:** Atomically adds each matrix element to the corresponding element
  of `Res` or `Arr` using `ComponentTy` arithmetic. `groupshared` data is
  bit-cast to `ComponentTy` before addition.
- **Synchronization:** For `Wave` and `ThreadGroup` scope, the call occurs in
  control flow uniform at `Scope`.
- **Notes:** 16-byte alignment is recommended for `Wave` and `ThreadGroup`
  scope buffer destinations.

###### `Matrix::MultiplyAccumulate`

```c++
template <ComponentEnum LHSTy, ComponentEnum RHSTy, SIZE_TYPE K,
          MatrixUseEnum UseLocal = Use>
typename hlsl::enable_if<Use == MatrixUse::Accumulator && UseLocal == Use,
                         void>::type
Matrix::MultiplyAccumulate(
    const Matrix<LHSTy, M, K, MatrixUse::A, Scope> MatrixA,
    const Matrix<RHSTy, K, N, MatrixUse::B, Scope> MatrixB);
```

- **Requires:** `Use` is `Accumulator`.
- **Requires:** `Scope` is `Wave` or `ThreadGroup`.
- **Effects:** Replaces the object with
  $\mathtt{this} + (\mathtt{MatrixA} \times \mathtt{MatrixB})$.
- **Synchronization:** The call occurs in control flow uniform at `Scope`.

###### `Matrix::Accumulate`

```c++
template <ComponentEnum CompTy, MatrixUseEnum UseLocal = Use>
typename hlsl::enable_if<Use == MatrixUse::Accumulator && UseLocal == Use,
                         void>::type
Matrix::Accumulate(const Matrix<CompTy, M, N, MatrixUse::A, Scope> MatrixA);

template <ComponentEnum CompTy, MatrixUseEnum UseLocal = Use>
typename hlsl::enable_if<Use == MatrixUse::Accumulator && UseLocal == Use,
                         void>::type
Matrix::Accumulate(const Matrix<CompTy, M, N, MatrixUse::B, Scope> MatrixB);
```

- **Requires:** `Use` is `Accumulator`.
- **Requires:** `Scope` is `Wave` or `ThreadGroup`.
- **Effects:** Adds the argument matrix element-wise to the object.
- **Synchronization:** The call occurs in control flow uniform at `Scope`.

###### `linalg::InterlockedAccumulate`

```c++
template <uint Align = 64, typename InputElTy, SIZE_TYPE M>
typename hlsl::enable_if<hlsl::is_arithmetic<InputElTy>::value, void>::type
linalg::InterlockedAccumulate(RWByteAddressBuffer Res,
                              uint StartOffset, vector<InputElTy, M> Vec);
```

- **Requires:** The address of `Res` at `StartOffset` is aligned to at least 64
  bytes.
- **Effects:** Atomically adds each element of `Vec` to the corresponding
  element of `Res` using `InputElTy` arithmetic.

###### `linalg::AccumulatorLayout`

```c++
MatrixUseEnum linalg::AccumulatorLayout();
```

- **Returns:** `MatrixUse::A` or `MatrixUse::B`, identifying the
  implementation's accumulator layout.
- **Remarks:** The driver compiler evaluates the result as a compile-time
  constant.

###### `linalg::Multiply(Matrix, Matrix)`

```c++
template <ComponentEnum OutTy, ComponentEnum ATy, ComponentEnum BTy,
          SIZE_TYPE M, SIZE_TYPE N, SIZE_TYPE K>
Matrix<OutTy, M, N, MatrixUse::Accumulator, MatrixScope::Wave> linalg::Multiply(
    const Matrix<ATy, M, K, MatrixUse::A, MatrixScope::Wave> MatrixA,
    const Matrix<BTy, K, N, MatrixUse::B, MatrixScope::Wave> MatrixB);

template <ComponentEnum CompTy, SIZE_TYPE M, SIZE_TYPE N, SIZE_TYPE K>
Matrix<CompTy, M, N, MatrixUse::Accumulator, MatrixScope::Wave>
linalg::Multiply(
    const Matrix<CompTy, M, K, MatrixUse::A, MatrixScope::Wave> MatrixA,
    const Matrix<CompTy, K, N, MatrixUse::B, MatrixScope::Wave> MatrixB);

template <ComponentEnum OutTy, ComponentEnum ATy, ComponentEnum BTy,
          SIZE_TYPE M, SIZE_TYPE N, SIZE_TYPE K>
Matrix<OutTy, M, N, MatrixUse::Accumulator, MatrixScope::ThreadGroup>
linalg::Multiply(
    const Matrix<ATy, M, K, MatrixUse::A, MatrixScope::ThreadGroup> MatrixA,
    const Matrix<BTy, K, N, MatrixUse::B, MatrixScope::ThreadGroup> MatrixB);

template <ComponentEnum CompTy, SIZE_TYPE M, SIZE_TYPE N, SIZE_TYPE K>
Matrix<CompTy, M, N, MatrixUse::Accumulator, MatrixScope::ThreadGroup>
linalg::Multiply(
    const Matrix<CompTy, M, K, MatrixUse::A, MatrixScope::ThreadGroup> MatrixA,
    const Matrix<CompTy, K, N, MatrixUse::B, MatrixScope::ThreadGroup> MatrixB);
```

- **Effects:** Multiplies `MatrixA` by `MatrixB`.
- **Synchronization:** The call occurs in control flow uniform at the matrices'
  scope.
- **Returns:** An $M \times N$ `Accumulator` matrix containing the product. The
  output component type is `OutTy` when specified and `CompTy` otherwise.

###### `linalg::Multiply(Matrix, vector)`

``` c++
template <typename OutputElTy, typename InputElTy, SIZE_TYPE M, SIZE_TYPE K,
          ComponentEnum MatrixDT>
typename hlsl::enable_if<hlsl::is_arithmetic<InputElTy>::value,
                         vector<OutputElTy, M>>::type
linalg::Multiply(
    Matrix<MatrixDT, M, K, MatrixUse::A, MatrixScope::Thread> MatrixA,
    vector<InputElTy, K> Vec);

template <typename OutputElTy, typename InputElTy, ComponentEnum InputInterp,
          SIZE_TYPE M, SIZE_TYPE K, SIZE_TYPE VecK, ComponentEnum MatrixDT>
typename hlsl::enable_if<
    InterpretedVector<InputElTy, VecK, InputInterp>::Size == K,
    vector<OutputElTy, M>>::type
linalg::Multiply(
    Matrix<MatrixDT, M, K, MatrixUse::A, MatrixScope::Thread> MatrixA,
    InterpretedVector<InputElTy, VecK, InputInterp> InterpVec);

```

- **Requires:** For the `InterpretedVector` overload,
  `InterpretedVector<InputElTy, VecK, InputInterp>::Size == K`.
- **Effects:** Multiplies `MatrixA` by the input column vector.
- **Returns:** The $M$-element product converted to `OutputElTy`.
- **Notes:** The function may be called from divergent control flow.

###### `linalg::OuterProduct`

```c++
template <ComponentEnum OutTy, typename InputElTy, SIZE_TYPE M, SIZE_TYPE N>
typename hlsl::enable_if<
    hlsl::is_arithmetic<InputElTy>::value,
    Matrix<OutTy, M, N, MatrixUse::Accumulator, MatrixScope::Thread>>::type
linalg::OuterProduct(vector<InputElTy, M> VecA, vector<InputElTy, N> VecB);
```

- **Effects:** Computes the outer product of `VecA` and `VecB`.
- **Returns:** An $M \times N$ thread-scope `Accumulator` matrix with component
  type `OutTy`.

###### `linalg::MultiplyAdd`

``` c++
template <typename OutputElTy, typename InputElTy, typename BiasElTy,
          SIZE_TYPE M, SIZE_TYPE K, ComponentEnum MatrixDT>
typename hlsl::enable_if<hlsl::is_arithmetic<InputElTy>::value &&
                             hlsl::is_arithmetic<BiasElTy>::value,
                         vector<OutputElTy, M>>::type
linalg::MultiplyAdd(
    Matrix<MatrixDT, M, K, MatrixUse::A, MatrixScope::Thread> MatrixA,
    vector<InputElTy, K> Vec, vector<BiasElTy, M> Bias);

template <typename OutputElTy, typename InputElTy, ComponentEnum InputInterp,
          typename BiasElTy, SIZE_TYPE M, SIZE_TYPE K, SIZE_TYPE VecK,
          ComponentEnum MatrixDT>
typename hlsl::enable_if<
    VecK == __detail::ScalarCountFromPackedComponents<InputInterp, K>::Value &&
        hlsl::is_arithmetic<BiasElTy>::value,
    vector<OutputElTy, M>>::type
linalg::MultiplyAdd(
    Matrix<MatrixDT, M, K, MatrixUse::A, MatrixScope::Thread> MatrixA,
    InterpretedVector<InputElTy, VecK, InputInterp> InterpVec,
    vector<BiasElTy, M> Bias);

template <typename OutputElTy, typename InputElTy, ComponentEnum BiasElTy,
          SIZE_TYPE M, SIZE_TYPE K, ComponentEnum MatrixDT>
typename hlsl::enable_if<hlsl::is_arithmetic<InputElTy>::value,
                         vector<OutputElTy, M>>::type
linalg::MultiplyAdd(
    Matrix<MatrixDT, M, K, MatrixUse::A, MatrixScope::Thread> MatrixA,
    vector<InputElTy, K> Vec, VectorRef<BiasElTy, M> BiasRef);

template <typename OutputElTy, typename InputElTy, ComponentEnum InputInterp,
          ComponentEnum BiasElTy, SIZE_TYPE M, SIZE_TYPE K, SIZE_TYPE VecK,
          ComponentEnum MatrixDT>
typename hlsl::enable_if<
    InterpretedVector<InputElTy, VecK, InputInterp>::Size == K,
    vector<OutputElTy, M>>::type
linalg::MultiplyAdd(
    Matrix<MatrixDT, M, K, MatrixUse::A, MatrixScope::Thread> MatrixA,
    InterpretedVector<InputElTy, VecK, InputInterp> InterpVec,
    VectorRef<BiasElTy, M> BiasRef);
```

- **Requires:** For an `InterpretedVector` argument, its interpreted element
  count is `K`.
- **Requires:** A native bias is an arithmetic vector with `M` elements.
- **Effects:** Multiplies `MatrixA` by the input column vector, converts the bias
  to `OutputElTy`, and adds it to the product. A `VectorRef` bias is read from
  memory.
- **Returns:** The $M$-element result converted to `OutputElTy`.
- **Notes:** The function may be called from divergent control flow.
- **Notes:** `VectorRef` permits the implementation to combine bias loads with
  arithmetic.

##### HLSL to DXIL operation mapping

| HLSL operation | DXIL operation |
|---|---|
| `Matrix::Splat` | `linAlgFillMatrix` |
| `Matrix::Cast` | `linAlgCopyConvertMatrix` |
| `Matrix::Load` from a buffer | `linAlgMatrixLoadFromDescriptor` |
| `Matrix::Load` from `groupshared` memory | `linAlgMatrixLoadFromMemory` |
| `Matrix::Length` | `linAlgMatrixLength` |
| `Matrix::GetCoordinate` | `linAlgMatrixGetCoordinate` |
| `Matrix::Get` | `linAlgMatrixGetElement` |
| `Matrix::Set` | `linAlgMatrixSetElement` |
| `Matrix::Store` to a buffer | `linAlgMatrixStoreToDescriptor` |
| `Matrix::Store` to `groupshared` memory | `linAlgMatrixStoreToMemory` |
| `Matrix::Accumulate` | `linAlgMatrixAccumulate` |
| `Matrix::MultiplyAccumulate` | `linAlgMatrixMultiplyAccumulate` |
| `Matrix::InterlockedAccumulate` to a buffer | `linAlgMatrixAccumulateToDescriptor` |
| `Matrix::InterlockedAccumulate` to `groupshared` memory | `linAlgMatrixAccumulateToMemory` |
| `linalg::AccumulatorLayout` | `linAlgMatrixQueryAccumulatorLayout` |
| `linalg::Multiply(Matrix, Matrix)` | `linAlgMatrixMultiply` |
| `linalg::Multiply(Matrix, vector)` | `linAlgMatVecMul` |
| `linalg::MultiplyAdd` | `linAlgMatVecMulAdd` |
| `linalg::OuterProduct` | `linAlgMatrixOuterProduct` |
| `linalg::Convert` | `linAlgConvert` |
| `linalg::InterlockedAccumulate` | `linAlgVectorAccumulateToDescriptor` |

### Interchange Format Additions

#### DXIL Types

This feature adds the following new DXIL enumerations, which used as immediate
arguments to the new operations.

```c++
namespace DXIL {
enum class MatrixUse : uint32_t {
  A = 0,
  B = 1,
  Accumulator = 2,
};

enum class UniformityScope : uint32_t {
  Thread = 0,
  Wave = 1,
  ThreadGroup = 2,
};

enum class ComponentType : uint32_t {
  Invalid = 0,
  I1 = 1,
  I16 = 2,
  U16 = 3,
  I32 = 4,
  U32 = 5,
  I64 = 6,
  U64 = 7,
  F16 = 8,
  F32 = 9,
  F64 = 10,
  SNormF16 = 11,
  UNormF16 = 12,
  SNormF32 = 13,
  UNormF32 = 14,
  SNormF64 = 15,
  UNormF64 = 16,
  PackedS8x32 = 17,
  PackedU8x32 = 18,

  // BEGIN NEW FOR SM 6.10
  I8 = 19,
  U8 = 20,
  F8_E4M3FN = 21,
  F8_E5M2 = 22,
  BFloat16 = 23,
  // END

  LastEntry
};

} // namespace dxil
```

The compiler will generate a permutation of typed matrix handles with names of
the format `%dx.types.LinAlgMatrix<mangling>`. The mangling scheme for
each type name will capture the type parameterization with the tokens `C`,
`M`, `N`, `U` and `S` denoting each encoded property.

```
  ; Matrix<ComponentType::F16, 16, 16, MatrixUse::A, MatrixScope::Wave>
  %dx.types.LinAlgMatrixC8M16N16U0S1    = type { i8 * }
  ; Matrix<ComponentType::F16, 16, 16, MatrixUse::B, MatrixScope::Wave>
  %dx.types.LinAlgMatrixC8M16N16U1S1    = type { i8 * }
  ; Matrix<ComponentType::F32, 16, 16, MatrixUse::Accumulator, MatrixScope::Wave>
  %dx.types.LinAlgMatrixC9M16N16U2S1    = type { i8 * }
```

DXIL validation will enforce that a `LinAlgMatrix` of any type may not
be bitcast to any other type.

#### LinAlg Component Types

DXIL validation will enforce that `ComponentType` for matrix types must be one
of the valid linalg component types listed below:

* `ComponentType::I8`
* `ComponentType::I16`
* `ComponentType::I32`
* `ComponentType::I64`
* `ComponentType::U8`
* `ComponentType::U16`
* `ComponentType::U32`
* `ComponentType::U64`
* `ComponentType::F8_E4M3FN`
* `ComponentType::F8_E5M2`
* `ComponentType::F16`
* `ComponentType::BFloat16`
* `ComponentType::F32`
* `ComponentType::F64`

##### Type Metadata

A new named metadata `dx.targetTypes` will be added to contain mappings of
attributed matrix types to their type parameters avoiding needing to parse the
type mangling. For the given examples above metadata of the form below will be
generated:


```
!dx.targetTypes = !{!1, !2, !3}
; Matrix<ComponentType::F16, 16, 16, MatrixUse::A, MatrixScope::Wave>
!1 = !{%dx.types.LinAlgMatrixC8M16N16U0S1 undef, i32 8, i32 16, i32 16, i32 0, i32 1 }
; Matrix<ComponentType::F16, 16, 16, MatrixUse::B, MatrixScope::Wave>
!2 = !{%dx.types.LinAlgMatrixC8M16N16U1S1 undef, i32 8, i32 16, i32 16, i32 1, i32 1 }
; Matrix<ComponentType::F32, 16, 16, MatrixUse::Accumulator, MatrixScope::Wave>
!3 = !{%dx.types.LinAlgMatrixC9M16N16U2S1 undef, i32 9, i32 16, i32 16, i32 2, i32 1 }
```

> Note: to ease compatibility with modern LLVM we want the metadata to avoid
> encoding pointers since modern LLVM will convert pointers to opaque pointers
> losing the type information.

#### DXIL Operations

A new overload shape `[MatTy]` is introduced in the signatures below. This
shall be the `<mangling>` part of `%dx.types.LinAlgMatrix<mangling>` preceded
by the letter `m`.

##### `@dx.op.linAlgFillMatrix`

```llvm
declare %dx.types.LinAlgMatrix<mangling> @dx.op.linAlgFillMatrix.[MatTy].[TY](
  immarg i32,            ; opcode
  [Ty]                   ; fill value
  )
```

- **Validation:** The output matrix scope is `Wave` or `ThreadGroup`.
- **Effects:** Fills the matrix with the scalar value.
- **Returns:** The filled matrix.
- **Remarks:** If the scalar and matrix component types differ, the operation
  converts the scalar according to [Data Conversion Rules](#data-conversion-rules).

##### `@dx.op.linAlgCopyConvertMatrix`

```llvm
declare %dx.types.LinAlgMatrix<mangling> @dx.op.linAlgCopyConvertMatrix.[MatTy1].[MatTy2](
  immarg i32,                         ; opcode
  %dx.types.LinAlgMatrix<mangling>,   ; matrix source
  immarg i1                           ; transpose
  )
```

- **Validation:** Both matrix types have the same scope, which is `Wave` or
  `ThreadGroup`.
- **Validation:** If `transpose` is `0`, both matrix types have the same
  dimensions.
- **Validation:** If `transpose` is `1`, the dimensions of `MatTy1` are the
  transpose of the dimensions of `MatTy2`.
- **Effects:** Copies the source matrix, converts its component type and use to
  those of `MatTy1`, and transposes it when `transpose` is `1`.
- **Returns:** The converted matrix. The source matrix remains unchanged.

##### `@dx.op.linAlgMatrixLoadFromDescriptor`

```llvm
declare %dx.types.LinAlgMatrix<mangling> @dx.op.linAlgMatrixLoadFromDescriptor.[MatTy](
  immarg i32,            ; opcode
  %dx.types.Handle,      ; ByteAddressBuffer
  i32,                   ; Offset
  i32,                   ; Stride
  i32,                   ; matrix layout
  immarg i32             ; alignment
  )
```

- **Validation:** For `Wave` and `ThreadGroup` scope, `Layout` is `RowMajor` or
  `ColMajor`.
- **Validation:** `Stride` is `0` if `Layout` is neither `RowMajor` nor
  `ColMajor`.
- **Validation:** For `Thread` scope, the resource is an SRV
  `ByteAddressBuffer`.
- **Validation:** `alignment` is a multiple of 128 for `Thread` scope and 4
  otherwise.
- **Effects:** Loads the matrix from the resource.
- **Memory:** `Offset` and `Stride` are measured in bytes. Reads follow
  [Bounds Checking Behavior](#bounds-checking-behavior).
- **Returns:** The loaded matrix.

##### `@dx.op.linAlgMatrixLoadFromMemory`

```llvm
declare %dx.types.LinAlgMatrix<mangling> @dx.op.linAlgMatrixLoadFromMemory.[MatTy].[Ty](
  immarg i32,            ; opcode
  [Ty] addrspace(3)*,    ; groupshared Ty[M * N]
  i32,                   ; Offset
  i32,                   ; Stride
  i32                    ; matrix layout
  )
```

- **Validation:** The output matrix scope is `Wave` or `ThreadGroup`.
- **Validation:** If `[Ty]` is neither `i32` nor a vector of `i32`, its scalar
  type matches the matrix component type.
- **Validation:** If `[Ty]` is floating point, its scalar type matches the
  matrix component type.
- **Validation:** `Offset` is aligned to 4 bytes and `Stride` is a multiple of
  4 bytes.
- **Effects:** Loads the matrix from `groupshared` memory without conversion.
- **Memory:** `Offset` and `Stride` count matrix components, regardless of the
  storage type `[Ty]`.
- **Returns:** The loaded matrix.
- **Notes:** Sixteen-byte alignment for `Offset` and `Stride` may improve
  performance.

##### `@dx.op.linAlgMatrixLength`

```llvm
declare i32 @dx.op.linAlgMatrixLength.[MatTy](
  immarg i32,                        ; opcode
  %dx.types.LinAlgMatrix<mangling>   ; matrix
  )
```

- **Validation:** The matrix scope is `Wave` or `ThreadGroup`.
- **Returns:** The number of matrix elements stored by the active thread.

##### `@dx.op.linAlgMatrixGetCoordinate`

```llvm
declare <2 x i32> @dx.op.linAlgMatrixGetCoordinate.[MatTy](
  immarg i32,                         ; opcode
  %dx.types.LinAlgMatrix<mangling>,   ; matrix
  i32                                 ; thread-local index
  )
```

- **Validation:** The matrix scope is `Wave` or `ThreadGroup`.
- **Returns:** The coordinate for the thread-local index, with the row in `x`
  and the column in `y`.

##### `@dx.op.linAlgMatrixGetElement`

```llvm
declare [Ty] @dx.op.linAlgMatrixGetElement.[Ty].[MatTy](
  immarg i32,                         ; opcode
  %dx.types.LinAlgMatrix<mangling>,   ; matrix
  i32                                 ; thread-local index
  )
```

- **Validation:** The matrix scope is `Wave` or `ThreadGroup`.
- **Returns:** The element at the thread-local index, or `0` if the index is out
  of range for the active thread.

##### `@dx.op.linAlgMatrixSetElement`

```llvm
declare %dx.types.LinAlgMatrix<mangling> @dx.op.linAlgMatrixSetElement.[MatTy].[MatTy].[Ty](
  immarg i32,                         ; opcode
  %dx.types.LinAlgMatrix<mangling>,   ; input matrix
  i32,                                ; thread-local index
  [Ty]                                ; value
  )
```

- **Validation:** The matrix scope is `Wave` or `ThreadGroup`.
- **Effects:** If the thread-local index is in range, sets the corresponding
  element to `value`. Otherwise, the operation has no effect.
- **Returns:** The resulting matrix.

##### `@dx.op.linAlgMatrixStoreToDescriptor`

```llvm
declare void @dx.op.linAlgMatrixStoreToDescriptor.[MatTy](
  immarg i32,                         ; opcode
  %dx.types.LinAlgMatrix<mangling>,   ; matrix
  %dx.types.Handle,                   ; ByteAddressBuffer
  i32,                                ; Offset
  i32,                                ; Stride
  i32,                                ; matrix layout
  immarg i32                          ; alignment
  )
```

- **Validation:** `Layout` is `RowMajor` or `ColMajor`.
- **Validation:** The matrix scope is `Wave` or `ThreadGroup`.
- **Validation:** The resource is a UAV `RWByteAddressBuffer`.
- **Validation:** `alignment` is a multiple of 4.
- **Effects:** Stores the matrix to the resource.
- **Memory:** `Offset` and `Stride` are measured in bytes. Writes follow
  [Bounds Checking Behavior](#bounds-checking-behavior).

##### `@dx.op.linAlgMatrixStoreToMemory`

```llvm
declare void @dx.op.linAlgMatrixStoreToMemory.[MatTy].[Ty](
  immarg i32,                         ; opcode
  %dx.types.LinAlgMatrix<mangling>,   ; matrix
  [Ty] addrspace(3)*,                 ; groupshared Ty[M * N]
  i32,                                ; Offset
  i32,                                ; Stride
  i32                                 ; matrix layout
  )
```

- **Validation:** The matrix scope is `Wave` or `ThreadGroup`.
- **Validation:** The `groupshared` allocation is large enough for the write.
- **Validation:** If `[Ty]` is neither `i32` nor a vector of `i32`, its scalar
  type matches the matrix component type.
- **Validation:** If `[Ty]` is floating point, its scalar type matches the
  matrix component type.
- **Validation:** `Offset` is aligned to 4 bytes and `Stride` is a multiple of
  4 bytes.
- **Effects:** Stores the matrix to `groupshared` memory without conversion.
- **Memory:** `Offset` and `Stride` count matrix components, regardless of the
  storage type `[Ty]`.
- **Notes:** Sixteen-byte alignment for `Offset` and `Stride` may improve
  performance.

##### `@dx.op.linAlgMatrixQueryAccumulatorLayout`

```llvm
declare i32 @dx.op.linAlgMatrixQueryAccumulatorLayout(
  immarg i32             ; opcode
  )
```

- **Execution:** The driver evaluates and replaces this operation at compile
  time.
- **Returns:** `0` if accumulator matrices use `A` layout; otherwise, `1` for
  `B` layout.

##### `@dx.op.linAlgMatrixMultiply`

```llvm
declare %dx.types.LinAlgMatrix<mangling> @dx.op.linAlgMatrixMultiply.[MatTyC].[MatTyA].[MatTyB](
  immarg i32,                        ; opcode
  %dx.types.LinAlgMatrix<mangling>,  ; matrix A
  %dx.types.LinAlgMatrix<mangling>   ; matrix B
  )
```

- **Validation:** `MatTyA` has use `A`.
- **Validation:** `MatTyB` has use `B`.
- **Validation:** `MatTyC` has use `Accumulator`.
- **Validation:** All matrix types have the same scope, which is `Wave` or
  `ThreadGroup`.
- **Validation:** The dimensions of `MatTyA`, `MatTyB`, and `MatTyC` are
  $M \times K$, $K \times N$, and $M \times N$, respectively.
- **Validation:** The matrix component types are compatible.
- **Effects:** Computes $A \times B$.
- **Execution:** The operation occurs in control flow uniform at the matrix
  scope.
- **Returns:** The resulting accumulator matrix.

##### `@dx.op.linAlgMatrixAccumulate`

```llvm
declare %dx.types.LinAlgMatrix<mangling> @dx.op.linAlgMatrixAccumulate.[MatTyC].[MatTyLHS].[MatTyRHS](
  immarg i32,                         ; opcode
  %dx.types.LinAlgMatrix<mangling>,   ; matrix LHS
  %dx.types.LinAlgMatrix<mangling>    ; matrix RHS
  )
```

- **Validation:** `MatTyRHS` has use `A` or `B`.
- **Validation:** `MatTyLHS` and `MatTyC` have use `Accumulator` and are the
  same matrix type.
- **Validation:** All matrix types have the same scope, which is `Wave` or
  `ThreadGroup`.
- **Validation:** All matrix types have the same dimensions.
- **Validation:** The matrix component types are compatible.
- **Effects:** Computes $LHS + RHS$.
- **Execution:** The operation occurs in control flow uniform at the matrix
  scope.
- **Returns:** The resulting accumulator matrix.

##### `@dx.op.linAlgMatrixMultiplyAccumulate`

```llvm
declare %dx.types.LinAlgMatrix<mangling> @dx.op.linAlgMatrixMultiplyAccumulate.[MatTyR].[MatTyA].[MatTyB].[MatTyC](
  immarg i32,                        ; opcode
  %dx.types.LinAlgMatrix<mangling>,  ; matrix A
  %dx.types.LinAlgMatrix<mangling>,  ; matrix B
  %dx.types.LinAlgMatrix<mangling>   ; matrix C
  )
```

- **Validation:** `MatTyA` has use `A`.
- **Validation:** `MatTyB` has use `B`.
- **Validation:** `MatTyC` and `MatTyR` have use `Accumulator`.
- **Validation:** All matrix types have the same scope, which is `Wave` or
  `ThreadGroup`.
- **Validation:** The dimensions of `MatTyA` and `MatTyB` are $M \times K$ and
  $K \times N$, respectively.
- **Validation:** The dimensions of `MatTyC` and `MatTyR` are $M \times N$.
- **Validation:** The matrix component types are compatible.
- **Effects:** Computes $C + (A \times B)$.
- **Execution:** The operation occurs in control flow uniform at the matrix
  scope.
- **Returns:** The resulting accumulator matrix.

##### `@dx.op.linAlgMatVecMul`

``` llvm
declare <[NUMo] x [TYo]> @dx.op.linAlgMatVecMul.v[NUMo][TYo].[MatTy].v[NUMi][TYi](
  immarg i32,                        ; opcode
  %dx.types.LinAlgMatrix<mangling>,  ; matrix A
  immarg i1,                         ; is output signed
  <[NUMi] x [TYi]>,                  ; input vector
  immarg i32                         ; input interpretation type (DXIL::ComponentType)
  )
```

- **Validation:** The input vector length equals the matrix `K` dimension.
- **Validation:** The matrix has use `A` and scope `Thread`.
- **Validation:** The input interpretation type is a valid
  [LinAlg Component Type](#linalg-component-types).
- **Validation:** `is output signed` is `true` when `[TYo]` is a native
  floating-point type.
- **Effects:** Multiplies the matrix by the input column vector.
- **Returns:** The resulting output vector.

##### `@dx.op.linAlgMatVecMulAdd`

``` llvm
declare <[NUMo] x [TYo]> @dx.op.linAlgMatVecMulAdd.v[NUMo][TYo].[MatTy].v[NUMi][TYi].v[NUMo][TYb](
  immarg i32,                         ; opcode
  %dx.types.LinAlgMatrix<mangling>,   ; matrix A
  immarg i1,                          ; is output signed
  <[NUMi] x [TYi]>,                   ; input vector
  immarg i32,                         ; input interpretation type (DXIL::ComponentType)
  <[NUMo] x [TYb]>                   ; bias vector
  )
```

- **Validation:** The input vector length equals the matrix `K` dimension.
- **Validation:** The bias and output vector lengths equal the matrix `M`
  dimension.
- **Validation:** The bias and output vector element types match.
- **Validation:** The matrix has use `A` and scope `Thread`.
- **Validation:** The input interpretation type is a valid
  [LinAlg Component Type](#linalg-component-types).
- **Validation:** `is output signed` is `true` when `[TYo]` is a native
  floating-point type.
- **Effects:** Multiplies the matrix by the input column vector and adds the
  bias vector.
- **Returns:** The resulting output vector.

##### `@dx.op.linAlgMatrixAccumulateToDescriptor`

```llvm
declare void @dx.op.linAlgMatrixAccumulateToDescriptor.[MatTy](
  immarg i32,                         ; opcode
  %dx.types.LinAlgMatrix<mangling>,   ; matrix
  %dx.types.Handle,                   ; RWByteAddressBuffer
  i32,                                ; Offset
  i32,                                ; Stride
  i32,                                ; matrix layout
  immarg i32                          ; alignment
  )
```

- **Validation:** The matrix has use `Accumulator`.
- **Validation:** `Layout` is `OuterProductOptimal` for `Thread` scope.
- **Validation:** `Layout` is `RowMajor` or `ColMajor` for `Wave` and
  `ThreadGroup` scope.
- **Validation:** `Stride` is `0` if `Layout` is neither `RowMajor` nor
  `ColMajor`.
- **Validation:** The resource is a UAV `RWByteAddressBuffer`.
- **Validation:** `alignment` is a multiple of 128 for `Thread` scope and 4
  otherwise.
- **Effects:** Adds each matrix element to the corresponding resource element
  using the matrix component type.
- **Memory:** `Offset` and `Stride` are measured in bytes. Writes follow
  [Bounds Checking Behavior](#bounds-checking-behavior).

##### `@dx.op.linAlgMatrixAccumulateToMemory`

```llvm
declare void @dx.op.linAlgMatrixAccumulateToMemory.[MatTy].[Ty](
  immarg i32,                         ; opcode
  %dx.types.LinAlgMatrix<mangling>,   ; matrix
  [Ty] addrspace(3)*,                 ; groupshared Ty[M * N]
  i32,                                ; Offset
  i32,                                ; Stride
  i32                                 ; matrix layout
  )
```

- **Validation:** The matrix has use `Accumulator` and scope `Wave` or
  `ThreadGroup`.
- **Validation:** The `groupshared` allocation is large enough for the write.
- **Validation:** If `[Ty]` is neither `i32` nor a vector of `i32`, its scalar
  type matches the matrix component type.
- **Validation:** `Offset` is aligned to 4 bytes and `Stride` is a multiple of
  4 bytes.
- **Effects:** Adds each matrix element to the corresponding `groupshared`
  element using the matrix component type. Memory values are bit-cast to the
  matrix component type before addition.
- **Memory:** `Offset` and `Stride` count matrix components, regardless of the
  storage type `[Ty]`.
- **Notes:** Sixteen-byte alignment for `Offset` and `Stride` may improve
  performance.

##### `@dx.op.linAlgMatrixOuterProduct`

```llvm
declare %dx.types.LinAlgMatrix<mangling> @dx.op.linAlgMatrixOuterProduct.[MatTy].v[M][TY].v[N][TY](
  immarg i32,            ; opcode
  <[M] x [Ty]>,          ; vector A
  <[N] x [Ty]>           ; vector B
  )
```

- **Validation:** The matrix `M` dimension equals the length of vector `A`.
- **Validation:** The matrix `N` dimension equals the length of vector `B`.
- **Validation:** Vectors `A` and `B` have the same element type.
- **Validation:** The output matrix has scope `Thread`.
- **Effects:** Computes the outer product of vectors `A` and `B`.
- **Returns:** The resulting matrix.

##### `@dx.op.linAlgConvert`

```llvm
declare <[NUMo] x [TYo]> @dx.op.linAlgConvert.v[NUMo][TYo].v[NUMi][TYi](
  immarg i32,                         ; opcode
  <[NUMi] x [TYi]>,                   ; input vector
  immarg i32,                         ; input interpretation type (DXIL::ComponentType)
  immarg i32                          ; output interpretation type (DXIL::ComponentType)
  )
```

- **Validation:** If the input interpretation has a native DXIL scalar type,
  `[TYi]` is that type; otherwise, `[TYi]` is `i32`.
- **Validation:** If the output interpretation has a native DXIL scalar type,
  `[TYo]` is that type; otherwise, `[TYo]` is `i32`.
- **Validation:** `[NUMo]` equals `[NUMi]` multiplied by the input components
  per scalar and divided by the output components per scalar, rounded up.
- **Effects:** Converts the input vector according to
  [Data Conversion Rules](#data-conversion-rules).
- **Returns:** The converted vector with the output interpretation type.

##### `@dx.op.linAlgVectorAccumulateToDescriptor`

``` llvm
declare void @dx.op.linAlgVectorAccumulateToDescriptor.v[NUM][TY](
  immarg i32,       ; opcode
  %dx.types.Handle, ; destination RWByteAddressBuffer
  i32,              ; buffer offset
  immarg i32,       ; vector base alignment
  <[NUM] x [TY]>    ; input vector
  )
```

- **Validation:** The resource is a UAV `RWByteAddressBuffer`.
- **Validation:** `vector base alignment` is at least 64 bytes.
- **Effects:** Adds each vector element to the corresponding resource element
  using `[TY]` arithmetic.
- **Memory:** `buffer offset` is measured in bytes. Writes follow
  [Bounds Checking Behavior](#bounds-checking-behavior).

##### Data Conversion Rules

All APIs introduced in this specification which may apply conversions shall obey
these conversion rules.

If the source and destination types are integer types, and the destination type
can exactly represent the source value, the value is preserved; otherwise the
result is saturated.

> Note: this is different from the normal conversion rules for HLSL native data
> types!

If the source and destination types are floating point types, and the
destination type can exactly represent the source value, the result is the exact
value; otherwise the conversion is a best-approximation of the source value and
is implementation-defined.

If the source is an integer type and the destination is a floating point type
the result is a _round to nearest ties to even_ (RTNE) conversion.

If the source type is a floating point type and the destination is an integer
type the conversion is a _round to nearest ties to even_ (RTNE) saturating
conversion.

##### FP8 Types

This specification introduces two FP8 data formats which may be used with linear
algebra objects. They are `F8_E4M3FN` and `F8_E5M2`, and they identify floating
point formats composed of 8 bits with 4 exponent and 3 mantissa bits and 5
exponent and 2 mantissa bits respectively.

|               | E4M3FN (finite)   | E5M2               |
|---------------|-------------------|--------------------|
| Exponent Bias | 7                 | 15                 |
| Infinity      | N/A               | S.11111.00         |
| NaN           | S.1111.111        | S.11111.{01,10,11} |
| Zero          | S.0000.000        | S.00000.00         |
| Max           | S.1111.110 (448)  | S.11110.11 (57344) |
| Min           | S.0000.001 (2^-9) | S.00000.01 (2^-16) |

###### Emulating FP

The DirectX API specification requires that all implementations support both FP8
formats for matrices, bias, and input vectors. If the target hardware does not
support the used F8 type an implementation is allowed to convert to any other
floating point format as long as the destination format can accurately represent
all values of the format used in the shader's DXIL.

If the driver sees a conversion to an F8 type that is not supported, and the
result of that conversion is only used by linear algebra operations, the driver
may eliminate the conversion or replace it with a conversion to any other
floating point format as long as the new destination format can accurately
represent all values of the format used in the shader's DXIL.

> Note: Under emulation if a source value would be converted to a saturated
> infinity (see: [conversion rules](#data-conversion-rules)) when converting to
> an F8 type, but the source value can be represented accurately in the emulated
> FP type, this may cause expected behavior differences.

##### Bounds Checking Behavior

The `@dx.op.linAlgMatrixLoadFromDescriptor` operation loads data from a
descriptor. For load operations a default element value of zero casted to the
element type is substituted for out of bounds reads. An implementation may
either perform bounds checking on the full bounds of the load initializing the
full matrix to the default element value if any element is out of bounds, or it
may perform per-element bounds checking at a 4-byte granularity, initializing to
the default value elements that fall partially or entirely within an
out-of-bounds 4-byte memory block.

The `@dx.op.linAlgMatrixStoreToDescriptor`,
`@dx.op.linAlgMatrixAccumulateToDescriptor`, and
`@dx.op.linAlgVectorAccumulateToDescriptor` operations write data to a
descriptor. Writes to out of bounds memory are a no-op. An implementation may
either perform bounds checking on the full bounds of the store converting the
whole store to a no-op if any element is out of bounds, or it may perform
per-element bounds checking at a 4-byte granularity, converting to no-ops the
stores of elements that fall partially or entirely within an out-of-bounds
4-byte memory block.

> Note: bounds checking is not required for reads and writes to root descriptors
> as D3D does not attach dimensions to root descriptors.

##### Pipeline State Validation Metadata

Shader Model 6.10 will introduce a version 4 of the Pipeline
State Validation RuntimeInfo structure. A new 32-bit unsigned integer
`LinalgMatrixUses` will count the number of `MatrixUse` objects appended after
the signature output vectors (presently the last data at the end of `PSV0` for
version 3).

The `MatrixUse` object is defined:

```c
struct MatrixUse {
  uint32_t Dimensions[3]; // M, N, K
  uint8_t Scope;
  uint8_t OperandType;
  uint8_t ResultType;
  uint8_t RESERVED; // unused but reserved for padding/alignment.
  uint32_t Flags;
};
```

This object will encode each matrix shape and element type as used by the DXIL
operations in the `linAlgMatrixMultiply`, `linAlgMatrixMultiplyAccumulate` and
`linAlgMatVecMulAdd` opcode classes.

The `Scope` field will encode one of the values defined by
[`DXIL::UniformityScope`](#dxil-types).

The `OperandType` and `ResultType` fields will encode values from
[`DXIL::ComponentType`](#dxil-types).

> Open questions:
> 1) Do we need the M and N dimensions or just the K dimension?
> 2) Do we need both operand types, or should we expect the operands to be the
>    same type?
> 3) Is the `Flags` field required, and what does each bit represent?
> 4) Does `linAlgMatrixLoadFromDescriptor` require a source-format operand, or
>    does the matrix component type determine the source format?
> 5) Should `UniformityScope` reserve a value for quad scope?

### Device Capability

The HLSL feature targets Shader Model 6.10. Device support is exposed through
`D3D12_LINEAR_ALGEBRA_TIER`, granular operation queries, and native-operation
enumeration as specified under [D3D API Additions](#d3d-api-additions).
Pipeline creation validates the shader's linear algebra PSV metadata against
these capabilities. Implementations may emulate only the combinations and
precision behavior explicitly permitted by the reported support flags and Tier
1 requirements.

### Dependencies, Risks, and Open Questions

This proposal depends on coordinated SM 6.10 support in DXC and the DXIL
validator, PSV metadata version 4, D3D12 runtime and debug-layer support, DDI
version `D3D12DDI_FEATURE_VERSION_LINEAR_ALGEBRA_0115_2`, SDK headers, WARP,
PIX, conformance tests, and driver implementations.

The primary compatibility risk is disagreement between shader metadata,
granular capability answers, and enumerated supported configurations. Runtime and
debug-layer validation must keep those representations consistent. Emulated
FP8 inputs and reduced-precision outputs can also produce observable numerical
differences within the allowances specified here.

Before Review, the proposal must name sponsors and required reviewers, assign
implementation owners and delivery tracking, finish the WARP and PIX plans,
and resolve the open PSV metadata questions recorded in the interchange-format
section.

## Testing

DXC tests must cover every HLSL overload, scope and use restriction, component
type, dimension limit, conversion path, control-flow requirement, and expected
diagnostic. Code-generation tests must verify each DXIL operation signature,
matrix type and `dx.targetTypes` entry, and every PSV linear algebra record.

DXIL validator tests must cover valid combinations and reject malformed types,
dimensions, scopes, layouts, alignments, resource kinds, and operation
signatures. Runtime and debug-layer tests must exercise Tier 1, granular and
enumeration query consistency, `_0115_1` translation, pipeline validation,
resource alignment, matrix conversion, barriers, predication, and unsupported
configurations.

Execution and conformance tests must compare matrix multiply,
multiply-accumulate, vector-matrix multiply-add, conversion, outer product,
loads and stores, and atomic accumulation across WARP and at least two vendor
implementations. Numerical tests must include saturation, RTNE conversion,
FP8 emulation, permitted accumulation precision, bounds behavior, and
transposed layouts.

## Alternatives considered

The design supersedes the DirectX Wave Matrix preview and unifies it with the
cross-platform direction established by Vulkan cooperative matrices. Keeping
separate HLSL and D3D proposals was also considered in practice, but a unified
document makes the language, interchange, runtime capability, validation, and
DDI contracts reviewable as one feature.

## Acknowledgments

A big thank you to all of the contributing authors of this specification, but in
particular to [Ashley Coleman](https://github.com/V-FEXrt) who drove much of the
implementation in DXC, and [Jack Elliott](https://github.com/JoeCitizen), who
drove the conformance testing and worked with [Jesse
Natalie](https://github.com/jenatali) on the D3D API specification.

This was a huge feature with a lot of moving parts and took a monumental effort
to get here.

Also thank you to [Jeff Bolz](https://github.com/jeffbolznv), [Alan
Baker](https://github.com/alan-baker), [David Neto](https://github.com/dneto0),
and [Justin Stoecker](https://github.com/jstoecker) for all their feedback and
patience.


