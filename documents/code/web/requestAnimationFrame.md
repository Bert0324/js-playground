# `requestAnimationFrame`

## Why we need `requestAnimationFrame`

When we want to implement some of animation in the browser, the first thing in our mind must be CSS, like `animation` with `@keyframes`, or `transition`.

But for the tasks whose animation is driven by complex data structure, JS is still the only choice.

There is an example in [fiddle](https://jsfiddle.net/y6xshd3e/), we can see the animation will be stuck while the main thread is being used, such as scripting or rendering.

Under this circumstance, `requestAnimationFrame` can be a good choice.

There is a solution via `requestAnimationFrame` in [fiddle](https://jsfiddle.net/xe7oqfjs/1/).

Compared to the first animation, The second animation is smoother, the reason is that `setTimeout` cannot make sure the task can be executed within specific interval while there is a task occupying the main thread.

## How `requestAnimationFrame` works

`requestAnimationFrame` will request that your animation function be called before the browser performs the next repaint. 

The number of callbacks is usually 60 times per second, but will generally match the display refresh rate in most web browsers as per W3C recommendation.

In this way, `requestAnimationFrame`'s interval is more stable, and aim to animation, browsers do more optimization via `requestAnimationFrame`.

## use `requestAnimationFrame` to calculate FPS

As `requestAnimationFrame` is called between two repaint time point, in this way, it can be used to calculate FPS.

```js
const getFpsAtTimePoint = () => {
  return new Promise((resolve) => {
    requestAnimationFrame((t1) => {
      requestAnimationFrame((t2) => {
        resolve(Math.round(1000 / (t2 - t1)));
      });
    });
  });
};

const getFpsWithinTheSecond = () => new Promise(resolve => {
	let repaint = 0;
	const start = performance.now();
	const withRepaint = () => {
		requestAnimationFrame(() => {
			if ((performance.now() - start) < 1000) {
				repaint += 1;
				withRepaint();
			} else {
				resolve(repaint);
			}
		});
	};
	withRepaint();
});

const createFpsHandler = () => {
	let repaint;
	let stop;
	let ret;
	let start;
	const init = () => {
		ret = undefined;
		stop = false;
		repaint = 0;
		start = performance.now();
	};
	init();
	const withRepaint = () => {
		requestAnimationFrame(() => {
			if (!stop) {
				repaint += 1;
				withRepaint();
			}
		});
	};
	return {
		start: () => {
			init();
			withRepaint();
		},
		end: () => {
			stop = true;
			if (!ret) ret = repaint / ((performance.now() - start) / 1000);
			return ret;
		}
	}
};
```

## Reference

- <https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame>
