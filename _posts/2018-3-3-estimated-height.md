---
layout: post
title: Weird bug with touch events in a UITableView with dynamic cell heights
---

Recently I solved an interesting iOS bug where the controls inside table view cells seemed to be receiving touch events in the wrong frame. Here's a clip of the bug in action:

![Touch events translated below the control gives an illusion of touches on one slider causing a slider in an above cell to move instead]({{ "/assets/2018-3-3-clip1.gif" | absolute_url }})

In the clip, a user is touches one slider but a different slider in a cell above moves.

Initially, I thought the cell wasn't being properly recycled, or the view model was mapped to the wrong index path. Instead, the correct slider was being controlled, but it was being controlled from touches in the view that were offset from its displayed frame on the screen. Furthermore, once the table view was reloaded, things returned to normal and touches were registered in the correct frame.

Because the cell heights were dynamic, I had to override `tableView:heightForRowAt:`, and I also implemented `tableView:estimatedHeightForRowAt:`. The cells had constraint-based animations to expand when they were selected, and collapse when deselected. Here were the original implementations:

```swift
func tableView(_ tableView: UITableView, estimatedHeightForRowAt indexPath: IndexPath) -> CGFloat {
  let viewModel = lightViewModels[indexPath.row]
  
  if viewModel.isDimmer {
    return SceneLightDimmerCell.estimatedHeight // 140
  } else {
    return SceneLightSwitchCell.estimatedHeight // 96
  }
}

func tableView(_ tableView: UITableView, heightForRowAt indexPath: IndexPath) -> CGFloat {
  let viewModel = lightViewModels[indexPath.row]
  
  if viewModel.isDimmer {
    return SceneLightDimmerCell.heightWhenSelected(isRowSelected(at: indexPath)) // ? 140 : 96
  } else {
    return SceneLightSwitchCell.heightWhenSelected(isRowSelected(at: indexPath)) // ? 96 : 44
  }
}
```

For `tableView:estimatedHeightForRowAt:`, I provided a "simpler" implementation, which did not account for the selection. This was exactly the problem. When the table view is first drawn, it uses the estimate to optimize table drawing. From [the Apple Documentation](https://developer.apple.com/documentation/uikit/uitableviewdelegate/1614926-tableview):

> Providing an estimate the height of rows can improve the user experience when loading the table view. If the table contains variable height rows, it might be expensive to calculate all their heights and so lead to a longer load time. Using estimation allows you to defer some of the cost of geometry calculation from load time to scrolling time.

Even though the table view appeared to have the correct layout, the touch coordinates were not mapping to the correct cells in the table view. However, this only happened if the table view had some selected (expanded) cells. Instead, they mapped to coordinates that would be correct *if all the cells were in the collapsed state*. Sure enough, deleting the `estimatedHeightForRow:atIndexPath:` implementation, solely relying on the `heightForRow:atIndexPath:`, resolved the issue:

The errors in the estimate implementation stacked up enough pixels in the y-dimension such that the mapping from touch coordinates to cell indexes was shifted.

![A side by side comparison of height and height estimate implementations]({{ "/assets/2018-3-3-diagram1.png" | absolute_url }})

I'm not sure if this bug is my fault, or Apple's fault, but I am sure that this is an example of premature optimization causing a costly issue. However, the mistake was an instructive lesson about how table views map touch and drawing coordinates to cells.

In conclusion, it's not a good idea to implement `estimatedHeightForRow:atIndexPath:` unless you actually need to optimize loading your table view. In this case, it was unnecessary because cell heights could be cheaply calculated. If calculating cell heights is expensive (e.g. loading rich content or calculating complex geometry), then it'd be worth estimating cell heights. But be wary that there can be side effects if your estimate is too inaccurate!
