import java.util.ArrayList;
import java.util.Optional;
import java.util.Random;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

class Data {
	final int num;

	Data(int n) {
		num = n;
	}

	String toJson() {
		return String.format("{\"num\":%d}", num);
	}

	static Optional<Data> fromJson(String json) {
		Matcher matcher = Pattern.compile("\\{\"num\":((-?)(\\d+))}").matcher(json);
		if (!matcher.find()) return Optional.empty();

		return Optional.of(new Data(Integer.parseInt(matcher.group(1))));
	}
}

class StreamLine<E> {
	private final ArrayList<E> queue = new ArrayList<>();
	private boolean halt = false;

	synchronized Optional<E> get() {
		try {
			while (queue.isEmpty()) {
				if (halt)
					return Optional.empty();
				else {
					wait();
				}
			}
			return Optional.of(queue.remove(queue.size() - 1));
		} catch (InterruptedException e) {
			throw new RuntimeException(e);
		}
	}

	synchronized void add(E e) {
		queue.add(e);
		notify();
	}

	synchronized void halt() {
		halt = true;
		notifyAll();
	}
}

class Producer implements Runnable {
	private final StreamLine<String> queue;
	private final int count;
	private final Random gen = new Random(System.currentTimeMillis());

	public Producer(StreamLine<String> queue, int count) {
		this.queue = queue;
		this.count = count;
	}

	public String produce() {
		Data data = new Data(gen.nextInt(100));
		if (data.num == 0) {
			return "error!";
		} else {
			return data.toJson();
		}
	}

	@Override
	public void run() {
		for (int i = 0; i < count; ++i)
			queue.add(produce());
		queue.halt();
	}
}

class Consumer implements Runnable {
	private final StreamLine<String> queue;
	public int ok = 0;
	public int error = 0;

	public Consumer(StreamLine<String> queue) {
		this.queue = queue;
	}

	public void consume(String json) {
		Optional<Data> res = Data.fromJson(json);
		if (res.isEmpty()) {
			error++;
		} else {
			ok++;
		}
	}

	@Override
	public void run() {

		while (true) {
			Optional<String> res = queue.get();
			if (res.isEmpty())
				return;
			consume(res.get());
		}
	}
}

public class Main {
	public static void main(String[] args) throws InterruptedException {
		StreamLine<String> queue = new StreamLine<>();

		Producer p = new Producer(queue, 10000);
		Consumer c1 = new Consumer(queue);
		Consumer c2 = new Consumer(queue);

		Thread t1 = new Thread(p);
		Thread t2 = new Thread(c1);
		Thread t3 = new Thread(c2);

		t1.start();
		t2.start();
		t3.start();

		t1.join();
		t2.join();
		t3.join();

		System.out.printf("c1: ok: %d, error: %d\nc2: ok: %d, error: %d\ntotal: %d",
				c1.ok, c1.error,
				c2.ok, c2.error,
				c1.ok + c1.error + c2.ok + c2.error);

	}
}

