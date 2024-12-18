# 协程逻辑

自己实现的协程逻辑 不依赖于Unity 可以手动更新
> 最开始是为帧同步实现的调用顺序可控的协程 虽然挺易用的 但后来还是被我从库中删除了 因为协程遍历的GC无法避免
```csharp
    public static class QCoroutine {
		private static QDelayList<IEnumerator> List { set; get; } = new QDelayList<IEnumerator>();
		public static bool IsRunning(this IEnumerator enumerator) {
			if (List.Contains(enumerator)) {
				return true;
			}
			return false;
		}
		public static IEnumerator Wait(Func<bool> func) {
			while (!func()) {
				yield return null;
			}
		}
		public static IEnumerator WaitCountZero<T>(this IList<T> List) {
			yield return Wait(() => List.Count == 0);
		}
		public static IEnumerator Start(this IEnumerator enumerator) {
			//QDebug.LogError("Start " + enumerator);
			if (UpdateIEnumerator(enumerator)) {
				List.Add(enumerator);
			}
			//else
			//{
			//	QDebug.LogError("Stop " + enumerator);
			//}
			return enumerator;
		}
		public const int RunImmediateMaxLoopTimes = 100;
		/// <summary>
		/// 以同步方式运行一次更新
		/// </summary>
		public static void RunImmediate(this IEnumerator enumerator) {
			for (int i = 0; i < RunImmediateMaxLoopTimes; i++) {
				if (!UpdateImmediate(enumerator)) {
					return;
				}
			}
			QDebug.LogError(enumerator + " 立即运行次数超过 " + RunImmediateMaxLoopTimes + " 次循环");
		}
		public static IEnumerator Start(this IEnumerator enumerator, List<IEnumerator> coroutineList) {
			IEnumerator ie = null;
			ie = enumerator.OnCallBack(() => coroutineList.Remove(ie));
			coroutineList.Add(ie);
			return ie.Start();
		}
		public static IEnumerator OnCallBack(this IEnumerator enumerator, Action callBack) {
			yield return enumerator;
			callBack?.Invoke();
		}
		public static void Stop(this IEnumerator enumerator) {
			List.Remove(enumerator);
			enumerator.RunImmediate();
		}
		static Dictionary<YieldInstruction, float> YieldInstructionList = new Dictionary<YieldInstruction, float>();
		/// <summary>
		/// 处理Unity内置的等待逻辑
		/// </summary>
		private static bool UpdateIEnumerator(YieldInstruction yieldInstruction) {
			if (yieldInstruction is WaitForSeconds waitForSeconds) {
				var m_Seconds = (float)waitForSeconds.GetValue("m_Seconds");
				if (!YieldInstructionList.ContainsKey(yieldInstruction)) {
					YieldInstructionList.Add(yieldInstruction, Time.time);
				}
				if (Time.time > YieldInstructionList[yieldInstruction] + m_Seconds) {
					YieldInstructionList.Remove(yieldInstruction);
					return false;
				}
				else {
					return true;
				}
			}
			else {
				Debug.LogError(nameof(QCoroutine) + "暂不支持 " + yieldInstruction + "\n" + QSerializeType.Get(yieldInstruction.GetType()));
				return false;
			}
		}
		private static bool UpdateImmediate(this IEnumerator enumerator) {
			var start = enumerator.Current;
			if (enumerator.Current is YieldInstruction ie) {
				//跳过处理YieldInstruction
			}
			else if (enumerator.Current is IEnumerator nextChild) {
				if (UpdateImmediate(nextChild)) {
					return true;
				}
			}
			enumerator.MoveNext();
			if (enumerator.Current != null && start != enumerator.Current) {
				return UpdateImmediate(enumerator);
			}
			else {
				return false;
			}
		}
		/// <summary>
		/// 更新迭代
		/// </summary>
		/// <param name="enumerator">迭代器</param>
		/// <returns>true继续等待 false时结束等待</returns>
		private static bool UpdateIEnumerator(IEnumerator enumerator) {
			var start = enumerator.Current;
			if (enumerator.Current is YieldInstruction ie) {
				if (UpdateIEnumerator(ie)) {
					return true;
				}
			}
			else if (enumerator.Current is IEnumerator nextChild) {
				if (UpdateIEnumerator(nextChild)) {
					return true;
				}
			}
			var result = enumerator.MoveNext();
			if (enumerator.Current != null && start != enumerator.Current) {
				return UpdateIEnumerator(enumerator);
			}
			else {
				return result;
			}
		}
		public static void Update() {
			List.Update();
			List.RemoveAll(ie => !UpdateIEnumerator(ie));
		}
		public static void StopAll() {
			List.Clear();
		}
	}
	public class QCoroutineQueue<T> {
		private Func<T, IEnumerator> action;
		private Queue<T> Queue { get; set; } = new Queue<T>();
		public int Count => Queue.Count;
		public bool IsRunning => Count > 0;
		public T Current => IsRunning ? Queue.Peek() : default;
		public QCoroutineQueue(Func<T, IEnumerator> action) {
			this.action = action;
		}
		private void Run(T value) {
			action(value).OnCallBack(() => {
				if (value.Equals(Current)) {
					Queue.Dequeue();
				}
				if (Queue.Count > 0) {
					Run(Queue.Peek());
				}
			}).Start();
		}
		public IEnumerator Start(T value) {
			Queue.Enqueue(value);
			if (Queue.Count == 1) {
				Run(value);
			}
			while (Queue.Count > 0) {
				yield return null;
			}
		}
		public void Clear() {
			Queue.Clear();
		}
	}
```