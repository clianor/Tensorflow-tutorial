# Tensorflow tf.data API start (2)

tensorflow에서 **tf.data API**는 단순할 뿐 아니라 재사용이 가능하고 복잡한 입력 파이프 라인도 구축할 수 있습니다.

에를들어 이미지 모델의 파이프 라인은 분산 파일 시스템의 파일에서 데이터를 들고온 후 각 이미지 데이터셋 섞고 배치를 적용하여 training 시스템이 적용할 수 있습니다.

- **tf.data.Dataset**는 각 요소가 하나 이상의 **Tensor**객체를 포함하는 요소들을 가지고 있습니다.
- 하나 이상의 **tf.Teneor**를 가지는 데이터 셋을 만들 수 있습니다. `(ex. Dataset.from_tensor_slices())`
- 데이터셋에 변환(transformation)을 실시 할 수 있고 변환(transformation)을 적용하면 변환된 **tf.data.Dataset** 객체들을 생성합니다. `(ex. Dataset.batch())`
- tf.data.Iterator는 데이터 집합에서 요소를 추출하는 주요 방법을 제공합니다. 반환 된 작업은 실행될 때 Iterator.get_next()의 다음 요소를 생성 Dataset하며 일반적으로 입력 파이프 라인 코드와 모델 간의 인터페이스 역할을합니다. 가장 간단한 iterator는 특정 one-shot iterator이고, 하나의 iterator를 Dataset반복합니다. 보다 정교한 용도 Iterator.initializer로이 작업을 사용하면 서로 다른 데이터 세트를 사용하는 반복기를 다시 초기화하고 매개 변수화 할 수 있으므로 동일한 프로그램에서 교육 및 유효성 검사 데이터를 여러 번 반복 할 수 있습니다.

## Basic mechanics

이 섹션에서는 **tf.data** 가 어떤식으로 작동하는지 알아 보겠습니다.

input pipeline을 사용하려면, 반드시 source를 정의 해야합니다. 예를 들어 메모리에 여러개의 tensor로 구성되어 있는 **Datasets**을 올리기 위해서는, **tf.data.Dataset.from_tensor()** 또는 **tf.data.Dataset.from_tensor_slice()** 를 이용하면 됩니다. 그리고 입력 데이터가 TFRecode 형태로 디스크에 저장되어 있으면 **tf.data.TFRecordDataset**를 사용하면 됩니다.

**Dataset** 객체를 정의가 되면은 이제 메소드들을 호출하여 **Dataset**을 여러가지형태로 변형을 할수가 있습니다. 예를들어 각 요소별로 변형이 가능하고 `Dataset.map` 또한 여러 요소들에 변형도 가능합니다. `Dataset.batch` 변형(transformation)과 관련된 많은 메소드들이 있는데 해당하는 메소드들의 리스트는 해당 링크를 확인하면 됩니다.  [tf.data transformation](https://www.tensorflow.org/api_docs/python/tf/data/Dataset)

Dataset을 사용하기 위해서는 **iterator**를 만들어줘야 되는데 데이터집합에 한요소씩 접근하기 위해서는 **Dataset.make_one_shot_iterator()**를 사용하면 됩니다. **tf.data.Iterator**는 두가지 기능을 제공하는데 **Iterator.initializer**를 이용하여 iterator의 상태를 초기화를 할 수 있고 **Iterator.get__next()**를 이용하면 tf.Tensor를 반환하고 다음 요소를 실행 할 수 있습니다.

### Dataset structure

```python
dataset1 = tf.data.Dataset.from_tensor_slices(tf.random_uniform([4, 10]))
print(dataset1.output_types)    # ==> tf.float32
print(dataset1.output_shapes)   # == > (10,)

dataset2 = tf.data.Dataset.from_tensor_slices(
    (tf.random_uniform([4]),
     tf.random_uniform([4, 100], maxval=100, dtype=tf.int32))
)
print(dataset2.output_types)    # ==> (tf.float32, tf.int32)
print(dataset2.output_shapes)   # ==> ((), (100, ))

dataset3 = tf.data.Dataset.zip((dataset1, dataset2))
print(dataset3.output_types)    # ==> (tf.float32, (tf.float32, tf.int32))
print(dataset3.output_shapes)   # ==> (10, ((), (100, )))
```

```python
dataset = tf.data.Dataset.from_tensor_slices(
    {
        'a': tf.random_uniform([4]),
        'b': tf.random_uniform([4, 100], maxval=100, dtype=tf.int32)
    }
)

print(dataset.output_type)      # ==> {'a' : tf.float32, 'b' : tf.int32}
print(dataset.output_shapes)    # ==> {'a' : () 'b' : (100, )}
```

### Create an iterator

Dataset에서 input date에 대해 표현을 하면, **Iterator**은 해 **Dataset** 세트의 요소에 엑세스하기 위해 작성하는 것입니다. **tf.data** API는 다음 반복자를 지원합니다.

- one-shot
- initializable
- reinitializable and
- feedable

one-shot 반복자는 명시적으로 초기화 할 필요없이, Dataset 통해 한 번 반복하는 지원 반복자의 간단한 형태이다. 원샷 반복자는 기존 큐 기반 입력 파이프 라인이 지원하는 거의 모든 경우를 처리하지만 매개 변수화를 지원하지 않습니다.

```python
dataset = tf.data.Dataset.range(100)
iterator = dataset.make_one_shot_iterator()
next_element = iterator.get_next()


print(sess.run(next_element))   # ==> 0
print(sess.run(next_element))   # ==> 1
print(sess.run(next_element))   # ==> 2
print(sess.run(next_element))   # ==> 3
```

initializable 반복자는 작업을 시작하기 전에 명시적으로 iterator.initializer를 실행하도록 요구합니다. 이 불편함을 감수하는 대신에 iterator를 초기화 할때 공급할 수 있는 하나 이상의 텐서를 사용하여 데이터 세트의 정의를 매개변수화 `tf.placeholder` 할 수 있습니다. 예제를 보면 확실히 알 수 있다.

```python
max_value = tf.placeholder(tf.int64, shape=[])
dataset = tf.data.Dataset.range(max_value)
iterator = dataset.make_initializable_iterator()
next_element = iterator.get_next()

# dataset의 element의 갯수를 10개로 초기화 한다.
sess.run(iterator.initializer, feed_dict={max_value: 10})
for _ in range(10):
    value = sess.run(next_element)
    print(value)    # ==> 0, 1, 2, 3, 4, .... , 9 (0부터 9까지)

# dataset의 element의 갯수를 100개로 초기화 한다.
sess.run(iterator.initializer, feed_dict={max_value: 100})
for _ in range(100):
    value = sess.run(next_element)
    print(value)    # ==> 0, 1, 2, 3, 4, .... , 100 (0부터 100까지)
```

reinitializable 반복자는 여러가지로 초기화를 할 수 있습니다. 예를 들어 일반화를 향상시키기 위해 입력 이미지의 랜덤으로 입력하는 train 위한 입력 파이프라인과 데이터가 얼마나 정확한지 확인하는 test를 위한 입력 파이프 라인은 Dataset의 구조를 동일한 하지만 서로 다른 객체를 사용해야 됩니다. 그때 필요한 것이 reinitializable 입니다.

```python
# training과 validation datasets는 같은 구조를 가지고 있습니다
training_dataset = tf.data.Dataset.range(100).map(
    lambda x: x + tf.random_uniform([], -10, 10, tf.int64))
validation_dataset = tf.data.Dataset.range(100)

# reinitializable iterator는 structure에 의해서 정의 됩니다.
# training_dataset 과 validation_dataset의 output_types과 output_shapes
# 속성이 호환 할 수 있습니다.
iterator = tf.data.Iterator.from_structure(training_dataset.output_types,
                                           training_dataset.output_shapes)
next_element = iterator.get_next()

training_init_op = iterator.make_initializer(training_dataset)
validation_init_op = iterator.make_initializer(validation_dataset)

# 20번을 반복하면서 train 과 validation 과정을 거친다.
for _ in range(20):
    # train dataset iterator를 초기화 한다.
    sess.run(training_init_op)
    for _ in range(100):
        print(sess.run(next_element))

    # validation dataset iterator를 초기화 한다.
    sess.run(validation_init_op)
    for _ in range(20):
        print(sess.run(next_element))
```

feedable 반복자는 tf.placeholder것을 선택하기 위해 

```python
training_dataset = tf.data.Dataset.range(100).map(
    lambda x: x + tf.random_uniform([[], -10, 10, tf.int64])).repeat()

validation_dataset = tf.data.Dataset.range(50)

# feedable iterator는 handle placeholder 와 구조로 정의됩니다.
# training_dataset 과 validation_dataset의 output_types과 output_shapes
# 속성이 호환 할 수 있습니다.
handle = tf.placeholder(tf.string, shape=[])
iterator = tf.data.Iterator.from_string_handle(
    handle, training_dataset.output_types, training_dataset.output_shapes)
next_element = iterator.get_next()

# feedable 반복자는 다양한 종류의 반복자와 함께 사용할 수 있습니다.
training_iterator = training_dataset.make_one_shot_iterator()
validation_iterator = validation_dataset.make_initializable_iterator()

# Iterator.string_handle () 메소드는 handle placeholder를 제공하기 위해
# 평가되고 사용될 수있는 텐서를 리턴합니다.
training_handle = sess.run(training_iterator.string_handle())
validation_handle = sess.run(validation_iterator.string_handle())

while True:
    for _ in range(200):
        print(sess.run(next_element, feed_dict={handle: training_handle}))

    # Run one pass over the validation dataset.
    sess.run(validation_iterator.initializer)
    for _ in range(50):
        print(sess.run(next_element, feed_dict={handle: validation_handle}))
```